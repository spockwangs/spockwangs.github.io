---
layout: post
title: How to support Unicode in C/C++
categories:
- programming language
---

To support multilanguage in programs we have to decide an encoding for
internal use.  It does not matter which encoding is used.  But it must be
able to encode all Unicode characters.  In this article I decide to use
wide characters as the internal encoding because it is easy to use and has
much support from standard C library.  The characters from the input is
converted to wide characters for internal use.  All the logic of the
program is based on the internal encoding.  When characters are required to
transmitted to outside (e.g. print to `stdout` or across the
network) they are converted to appropriate encoding.

## Locale

The functions in the C library which does convertions between multi-byte
characters and wide characters is locale-specific (particularly the
category `LC_CTYPE`).  Their behavior can be changed by
`setlocale` defined in `locale.h`.  So when we want to
convert from or to multi-byte characters we must set the value of
`LC_CTYPE `correctly.

There is a global locale in C++, as there is in C.  Initially the global
locale is the locale "C".  You can get current global locale by calling
<code>std::locale::locale()</code>.  Except the global locale you can
construct as many locales as you want and imbue each stream with a
different locale object.  Working with many different locales becomes easy
in C++, but in C you have to switch locales frequently.

## Input

Often the characters from the input source is encoded using some multi-byte
encoding like GBK or UTF-8.  We have to find a way to convert from these
multi-byte encoding to wide characters defined in C.  Foutunately C library
has provided many functions to do the conversion.  But we must take care to
set the relevant locale category to make them work correctly.

We can use `scanf` to convert the input multi-byte sequence to wide
characters, like the following which assmes the input characters are
encoded in GBK.

    wchar_t wcs[1024];
    setlocale(LC_CTYPE, "zh_CN.GBK");
    fscanf(fp, "%ls", wcs);

Or we can obtain the input as a byte-sequence and then call `mbstowcs` to do the conversion.

    char bytes[1024];
    int len;
    wchar_t *wcs;
    fscanf(fp, "%s", bytes);

    /* Get how many wide characters will be converted including NUL-terminator. */
    setlocale(LC_CTYPE, "zh_CN.GBK");
    len = mbstowcs(NULL, bytes, 0) + 1;
    wcs = (wchar_t *) malloc(len * sizeof(*wcs));
    mbstowcs(wcs, bytes, len);

But we must make sure the byte-sequence is a full valid multibyte sequence
or `-1` is returned by the call to `mbstowcs`.

In C++ we can use <code>wistream</code> to convert the input multibyte sequence to wide characters.

    wstring ws;  // define wide string to hold input
    wcin.sync_with_stdio(false);
    wcin.imbue(std::locale::locale("zh_CN.GBK"));
    wcin >> ws;

Due to some limitations with it <code>iostream::imbue()</code> does not
honor (but <code>fstream</code> does) the encoding we have to call
<code>ios::sync_with_stdio(false)</code> to enable conversion. (See <a
href="http://gcc.gnu.org/ml/libstdc++/2006-11/msg00058.html">http://gcc.gnu.org/ml/libstdc++/2006-11/msg00058.html</a>).

We can also obtain the input as a byte sequence and then convert it to the destination encoding.

    // Assume the input byte sequence is in 'from' with GBK encoding.
    string from;
    locale loc("zh_CN.GBK");
    const codecvt<wchar_t, char, mbstat_t>& conv =
    use_facet<codecvt<wchar_t, char, mbstat_t> >(loc);
    mbstat_t mystate;

    // Calculate how many characters there are in 'from'.
    int length = conv.length(mystate, from.c_str(), from.c_str()+from.length(),
    numeric_limits<size_t>::max());

    wchar_t *pws = new wchar_t[length+1];
    pws[length] = L'\0';
    const char *from_next;
    wchar_t *to_next;
    codecvt<wchar_t, char, mbstate_t>::result myresult  =
        conv.in(mystate, in.c_str(), in.c_str()+in.length(), from_next,
                pws, pws+length, to_next);
    if (myresult == codecvt<wchar_t, char, mbstate_t>::ok) {
        // Conversion is ok.
        ...
    }

## Output

When wide characters are printed to the standard output they must be
converted to multibyte sequence with the encoding specified by
`LC_CTYPE` environment variable.  This is easy to do with `setlocale`.

    /* Print wide characters in 'wcs'. */
    setlocale(LC_CTYPE, "");
    printf("%ls\n", wcs);

Or equivalently in C++:

    // Assume wide string is in 'ws'.
    ios::sync_with_stdio(false);
    wcout.imbue(locale(""));
    wcout << ws << endl;

If we want to print wide characters to other destinations (files or
network) we can convert the wide characters to multibyte sequence with the
expected encoding by set the category `LC_CTYPE`. For example the
following code convertes wide characters to multibyte sequence in GBK
encoding.

    /* Assume wide characters is in 'wcs'. */
    setlocale(LC_CTYPE, "zh_CN.GBK");
    int len = wcstombs(NULL, wcs, 0)+1;
    char *buf = (char *) malloc(len * sizeof(*buf));
    wcstombs(buf, wcs, len);

Or equivalently in C++:

    // Assume wide string is in 'ws'.
    locale loc("zh_CN.GBK");
    const codecvt<wchar_t, char, mbstate_t>&amp; conv =
    use_facet<codecvt<wchar_t, char, mbstate_t> >(loc);

    // At most 'len' bytes are required to store the string in
    // multibyte sequence.
    int len = conv.max_length() * (ws.length()+1);
    char *pstr = new char[len];
    const wchar_t *pwc;
    char *pc;
    mbstate_t mystate;
    codecvt<wchar_t, char, mbstate_t>::result myresult =
        conv.out(mystate, ws.c_str(), ws.c_str()+ws.length()+1, pwc,
                 pstr, pstr+ws.length()+1, pc);
    if (myresult == codecvt<wchar_t, char, mbstate_t>::ok) {
        // Conversion is ok.
        ...
    }

## Frequent tasks with wide characters and strings

The C library functions which handle wide characters are often prefixed
with `wcs`.  For the functions which handle `char` type
characters there is an equivalent function which handles wide characters.

## References

*  _C: A Reference Manual_, Fifth Edition.
*  [http://www.evanjones.ca/unicode-in-c.html](http://www.evanjones.ca/unicode-in-c.html)
*  [http://www.cl.cam.ac.uk/~mgk25/unicode.html](http://www.cl.cam.ac.uk/~mgk25/unicode.html)
