---
layout: post
title: "用Tensorflow C++ API训练模型"
date: 2018-01-13 21:14:48 +0800
comments: true
categories: 
---

用Tensorflow解决机器学习的问题时通常是用Python API构造模型并训练模型参数，然后将模型序列化到文件中。部署到线上时使用C++ API加载模型对输入进行预测。Tensorflow还专门提供了Serving模块来优化线上部署，除了进行预测外还提供了对模型的版本管理和模型热更新。但是为了更快地训练模型，缩短模型更新周期，就必须支持线上实时训练，也就是说必须支持用C++ API训练模型。但是C++ API目前还不完善，还不方便构造模型，所以仍然使用Python API构造模型，并导出给C++ API加载训练。

本文给出两种方法来编译C++库，一种是共享库，一种是静态库。
<!--more-->

## 准备

1. 安装Bazel
2. 安装Tensorflow for Python.
3. 下载Tensorflow源代码

   ``` shell
   $ git clone https://github.com/tensorflow/tensorflow.git
   ```
4. 用Python API构造模型，我们以MNIST的softmax回归模型为例，代码如下：

``` python create_model.py
import tensorflow as tf
import getopt
import sys
import exceptions
import traceback

class Usage(Exception):
    def __init__(self, msg):
        self.msg = msg

def usage(progname):
    print('''Usage:
    {progname} [-h | --help] --export_dir <dir> [--as_text]'''.format(progname=progname))
    
def main(argv=None):
    if argv is None:
        argv = sys.argv
    try:
        try:
            opts, args = getopt.getopt(argv[1:], "h", [ 'help', 'export_dir=', 'as_text' ])
        except getopt.error as msg:
            raise Usage(msg)

        global g_debug_mode
        export_dir = ''
        as_text = False
        for o, a in opts:
            if o in ('-h', '--help'):
                usage(argv[0])
                return 0
            elif o in ('-d'):
                g_debug_mode = True
            elif o in ('--export_dir'):
                export_dir = a
            elif o in ('--as_text'):
                as_text = True
            else:
                raise Usage('Bad option: %s' % (o))
        if len(export_dir) == 0:
            raise Usage('Bad value for option --export_dir')

        # Create the model
        x = tf.placeholder(tf.float32, [None, 784], name='x')
        W = tf.Variable(tf.zeros([784, 10]))
        b = tf.Variable(tf.zeros([10]))
        y = tf.matmul(x, W) + b

        # Define loss and optimizer
        y_ = tf.placeholder(tf.int32, [None], name='y_')
        cross_entropy = tf.losses.sparse_softmax_cross_entropy(labels=y_, logits=y)
        train_step = tf.train.GradientDescentOptimizer(0.5).minimize(cross_entropy, name='train_step')

        # Test trained model
        correct_prediction = tf.equal(tf.argmax(y, 1), tf.cast(y_, tf.int64))
        accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32), name='accuracy')

        # Export the computation graph.
        tf.train.write_graph(tf.get_default_graph().as_graph_def(), export_dir, 'mnist_graph.pb', as_text=as_text)
    except Usage as err:
        print(err.msg)
        usage(argv[0])
        return 1
    except Exception as e:
        traceback.print_exc()
        return 1

if __name__ == '__main__':
    main()
```

创建模型并导出到目录models.
```
$ create_model.py --export_dir=models
```

## 第一种方法：编译共享库

首先编译Tensorflow的C++共享库`libtensorflow_cc.so`.
```
$ cd tensorflow              # 进入tensorflow源代码目录
$ ./configure
$ bazel build --config=opt //tensorflow:libtensorflow_cc.so
```

下载并编译Tensorflow依赖的外部库。
```
$ cd tensorflow/contrib/makefile
$ ./build_all_linux.sh
```

安装头文件和共享库。
```
$ cp tensorflow
$ find tensorflow/core -name '*.h' | xargs -i cp --parents '{}' /usr/local/include/tf
$ cp -r bazel-genfiles/* /usr/local/include/tf
$ cp -r tensorflow/contrib/makefile/gen/protobuf/include/* /usr/local/include/tf
$ cp -r tensorflow/contrib/makefile/downloads/eigen/* /usr/local/include/tf
$ cp -r tensorflow/contrib/makefile/downloads/nsync/public/* /usr/local/include/tf
$ cp -r third_party /usr/local/include/tf
$ cp tensorflow/bazel-bin/tensorflow/libtensorflow_cc.so /usr/local/lib
$ cp tensorflow/bazel-bin/tensorflow/libtensorflow_framework.so /usr/local/lib
```

我们的C++训练代码如下。
```c++ train_model.cpp
#include <arpa/inet.h>
#include <string>
#include <vector>
#include <fstream>
#include "tensorflow/core/public/session.h"

using namespace std;

class DataSet {
public:
    DataSet()
    {}

    void LoadDataFromDir(const std::string& path)
    {
        const char* x_train_file = "train-images-idx3-ubyte";
        const char* y_train_file = "train-labels-idx1-ubyte";
        const char* x_test_file = "t10k-images-idx3-ubyte";
        const char* y_test_file = "t10k-labels-idx1-ubyte";
        m_x_train = ExtractImages(path + "/" + x_train_file);
        m_y_train = ExtractLabels(path + "/" + y_train_file);
        m_x_test = ExtractImages(path + "/" + x_test_file);
        m_y_test = ExtractLabels(path + "/" + y_test_file);
    }
        
    vector<double>& x_train()
    {
        return m_x_train;
    }

    vector<int>& y_train()
    {
        return m_y_train;
    }

    vector<double>& x_test()
    {
        return m_x_test;
    }

    vector<int>& y_test()
    {
        return m_y_test;
    }

private:
    uint32_t ReadUint32(ifstream& is)
    {
        uint32_t data = 0;
        if (is.read(reinterpret_cast<char*>(&data), 4)) {
            return ntohl(data);
        }
        throw logic_error("can't read 4 bytes");
    }
    
    uint8_t ReadUint8(ifstream& is)
    {
        uint8_t data = 0;
        if (is.read(reinterpret_cast<char*>(&data), 1)) {
            return data;
        }
        throw logic_error("can't read 1 byte");
    }
    
    vector<double> ExtractImages(const string& file)
    {
        ifstream is(file);
        if (!is) {
            throw logic_error("can't open file: " + file);
        }
        uint32_t magic = ReadUint32(is);
        if (magic != 2051) {
            throw logic_error("bad magic: " + to_string(magic));
        }
        uint32_t num = ReadUint32(is);
        uint32_t rows = ReadUint32(is);
        uint32_t cols = ReadUint32(is);
        vector<double> images;
        for (size_t i = 0; i < num*rows*cols; ++i) {
            uint8_t byte = ReadUint8(is);
            images.push_back(static_cast<double>(byte)/255.0);
        }
        return images;
    }

    vector<int> ExtractLabels(const string& file)
    {
        ifstream is(file);
        if (!is) {
            throw logic_error("can't open file: " + file);
        }
        uint32_t magic = ReadUint32(is);
        if (magic != 2049) {
            throw logic_error("bad magic: " + to_string(magic));
        }
        uint32_t num = ReadUint32(is);
        vector<int> labels;
        for (size_t i = 0; i < num; ++i) {
            uint8_t byte = ReadUint8(is);
            labels.push_back(byte);
        }
        return labels;
    }
    
    std::vector<double> m_x_train;
    std::vector<int> m_y_train;
    std::vector<double> m_x_test;
    std::vector<int> m_y_test;
};

using namespace tensorflow;

int main(int argc, char* argv[])
{
    try {
        string data_dir = "mnist_data";
        if (argc > 1) {
            data_dir = argv[1];
        }
        DataSet data_set;
        cout << "loading data from " << data_dir << " ... " << endl;
        data_set.LoadDataFromDir(data_dir);
        cout << "loading data ... done." << endl;
    
        // Initialize a tensorflow session
        Session* session;
        TF_CHECK_OK(NewSession(SessionOptions(), &session));

        // Read in the protobuf graph we exported
        // (The path seems to be relative to the cwd. Keep this in mind
        // when using `bazel run` since the cwd isn't where you call
        // `bazel run` but from inside a temp folder.)
        GraphDef graph_def;
        const char* graph_path = "mnist_graph.pb";
        if (argc > 2) {
            graph_path = argv[2];
        }
        cout << "reading graph from " << graph_path << " ... " << endl;
        TF_CHECK_OK(ReadBinaryProto(Env::Default(), graph_path, &graph_def));
        cout << "reading graph ... done." << endl;

        // Add the graph to the session
        TF_CHECK_OK(session->Create(graph_def));

        // Setup inputs and outputs:
        Tensor x_train(DataTypeToEnum<float>::v(), 
                       TensorShape{static_cast<int>(data_set.x_train().size())/784, 784});
        copy_n(data_set.x_train().begin(), data_set.x_train().size(), x_train.flat<float>().data());

        Tensor y_train(DataTypeToEnum<int>::v(), 
                       TensorShape{static_cast<int>(data_set.y_train().size())});
        copy_n(data_set.y_train().begin(), data_set.y_train().size(), y_train.flat<int>().data());

        std::vector<std::pair<string, tensorflow::Tensor>> inputs = {
            { "x", x_train },
            { "y_", y_train },
        };

        cout << "init global variables ... " << endl;
        TF_CHECK_OK(session->Run(inputs, { }, { "init" }, nullptr));
        cout << "init ... done." << endl;
     
        cout << "training ... " << endl;
        std::vector<tensorflow::Tensor> outputs;
        for (int i = 0; i < 100; ++i) {
            cout << "step " << i << endl;
            TF_CHECK_OK(session->Run(inputs, {}, { "train_step" }, nullptr));
        }
        cout << "training ... done." << endl;

        cout << "testing ... " << endl;
        Tensor x_test(DataTypeToEnum<float>::v(), 
                      TensorShape{static_cast<int>(data_set.x_test().size())/784, 784});
        copy_n(data_set.x_test().begin(), data_set.x_test().size(), x_test.flat<float>().data());

        Tensor y_test(DataTypeToEnum<int>::v(), 
                      TensorShape{static_cast<int>(data_set.y_test().size())});
        copy_n(data_set.y_test().begin(), data_set.y_test().size(), y_test.flat<int>().data());
        std::vector<std::pair<string, tensorflow::Tensor>> tests = {
            { "x", x_test },
            { "y_", y_test }
        };

        TF_CHECK_OK(session->Run(tests, { "accuracy" }, {}, &outputs));
        cout << "Accuracy: " << *outputs[0].scalar<float>().data() << endl;
        cout << "testing ... done." << endl;
    
        // Free any resources used by the session
        session->Close();
    } catch (exception& e) {
        cerr << e.what() << endl;
        return 1;
    }
    return 0;
}
```

编译训练代码。
```
$ g++ train_model.cpp -o train_model -I/usr/local/include/tf -ltensorflow_cc -ltensorflow_framework
```

下载[MNIST数据](http://yann.lecun.com/exdb/mnist/)到目录mnist_data并解压，然后运行训练代码。
```
$ ls mnist_data
t10k-images-idx3-ubyte
t10k-labels-idx1-ubyte
train-images-idx3-ubyte
train-labels-idx1-ubyte
$ ./train_model mnist_data models/mnist_graph.pb
loading data from mnist_data ... 
loading data ... done.
...
training ... done.
testing ... 
Accuracy: 0.9012
testing ... done.
```

## 第二种方法：编译静态库

``` shell
$ cd tensorflow/contrib/makefile
$ ./build_all_linux.sh
```
它的原理是利用bazel获取Tensorflow依赖的文件，然后编译链接成静态库。编译的Tensorflow静态库在`gen/lib/libtensorflow-core.a`. 同样使用上述C++代码，编译训练代码。
``` shell
$ cd tensorflow/contrib/makefile
$ g++ -std=c++11 train_model.cpp -o train_model \
-I ../../.. \
-I downloads/ \ 
-I downloads/eigen/ \
-I downloads/gemmlowp/ \
-I downloads/nsync/public/ \
-I downloads/fft2d/ 
-I gen/proto/ 
-I gen/proto_text/ 
-I gen/protobuf-host/include/ \
-Wl,--allow-multiple-definition \
-Wl,--whole-archive gen/lib/libtensorflow-core.a -Wl,--no-whole-archive \
downloads/nsync/builds/default.linux.c++11/nsync.a \
gen/protobuf-host/lib/libprotobuf.a -lz -lm -ldl -lpthread -lstdc++
```

运行模型：
``` shell
$ ./train_model mnist_data models/mnist_graph.pb
```
如果运行时报错说有些操作没有注册，这需要将相应的操作源代码文件（在`tensorflow/core/kernels/`下）放到`tf_op_files.txt`中重新编译。
