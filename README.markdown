This is a low-level jQuery plugin that allows you to parse the query strings commonly used in URLs.

It is designed to provide a solid base, on which fancier parsers can be easily implemented, but should also work out-of-the-box.

This originated from mkmanning's original jQuery.parseQuery plugin, though very little of the original remains; it is still released under the MIT license.

Comments, bug reports and feature requests are always welcome.


Documentation
=============

By default, `$.parseQuery()` takes the query parameter from the current page, and returns an Object with string keys and string values.

You can also pass a string, which will be used in place of the page's query string:

    $.parseQuery("?a=b&c=d") == {'a':'b','c':'d'}

When a key is specified multiple times, the last value is used:

    $.parseQuery("?a=b&a=c") == {'a':'c'}

When a key is specified without a value, the empty string is used:

    $.parseQuery("?a&b=c&=d&e=") == {'a':'','b':'c','':'d','e':''}

The strings are decoded, raw `'+'` is treated as `' '`, and %-encoding is assumed to be in CESU-8 (UTF-8 with optional surrogate pairs)

    $.parseQuery("?a+b=c%2Bd&%C3%A6=ae") == {'a b':'c+d','\u00e6':'ae'}

Values may contain any unescaped character except for `'&'` and `'%'`, and keys may contain any unescaped character except for `'&'`, `'%'` and `'='`. Both can contain encoded versions of these characters:

    $.parseQuery("?a=b=c&%3D=%26") == {'a':'b=c', '=':'&'}

If the %-escaping is mal-formed, or does not represent valid CESU-8, a URIError is thrown, internally using decodeURIComponent.

    $.parseQuery("?a=%")
    URIError: URI malformed.


Multi-valued Parameters
-----------------------

The encoding mechanism allows for keys in query strings to be associated with multiple values.

If you wish to specify that some keys have multiple values, you can provide an options Object with an "array_keys" key that is a regular expression, or a function:

    $.parseQuery({
        query: "a[]=b&a[]=c&d=e&d=f",
        array_keys: /\[\]$/
    }) == {'a[]':['b','c'], 'd':'f'}

If you want to set the array_keys parameter for all future calls to `$.parseQuery`, set `$.parseQuery.array_keys`.

    $.parseQuery.array_keys = /^(ids|names)$/;
    $.parseQuery({
        query: "ids=1&names=foo&names=bar"
    }) == {'ids':['1'],'names':['foo','bar']}

(A common idiom is for a parseQuery equivalent to guess which keys are multivalued by seeing which have multiple values - this heuristic makes it very easy to add bugs to your code, and is not supported)


Parameter Types and Advanced Customisation
------------------------------------------

Many people have previously implemented query string encoders and decoders, and as a result, it's likely that your web-application will use one that does not have identical semantics to this one.

Additionally, people often pass numbers or booleans in query parameters, and it would be nice to have all the parsing done centrally, instead of having to
implement type coercion everywhere.

To support both of these cases, users may pass a decode function to `$.parseQuery`, or set a global `$.parseQuery.decode` appropriately.

The function is called with two arguments, the input and the context, and always has `this` set to the current options.

The context is either undefined, which implies the input is a string representing a key. Or it is the previously decoded key, which implies the input is a string or null, representing the value.

To clarify, while parsing "?a&b=&c=d", decode will be called six times:

    decode(null, decode("a", null));
    decode("", decode("b", null));
    decode("d", decode("c", null));

When used in combination with array_keys, array_keys is tested against the output of decode with a null context. This gives you almost total control of how the output Object should be constructed.

For example:

    $.parseQuery({query: "id[]=1&id[]=2",
        array_keys: /^ids$/,
        decode: function (input, context) {
            input = this.default_decode(input);
            if (context === null) {
                return input.replace(/\[\]$/, 's')
            } else if (context === "ids") {
                input = parseInt(input);
                if (isNaN(input)) {
                    throw URIError("id was not a number");
                }
            }
            return input;
        }
    }) == {'ids':[1, 2]}

The advantage of having `this` set to the current options means that, not only can you access this.default_decode, you can also pass configuration to your decode function:

    $.parseQuery.decode = function (input, context) {
        input = this.default_decode(input);
        if(context && this.int_keys.test(context)) {
            input = parseInt(input);
        }
        return input;
    }
    $.parseQuery({
        int_keys: /_id$/,
        query: "a_id=1&b_id=2"
    }) == {'a_id':1, 'b_id':2}

Please stick to overriding `decode()` and leave `default_decode()` alone, if you want to make other decode functions available for each other, define them on `$.parseQuery`.


Semicolons
----------

The W3C recommends that query parsers treat semicolons as equivalent to ampersands. By default, we ignore this recommendation as the reasoning behind it is weak, and many people are unaware of it.

If you'd like to split on semicolons instead of ampersands, set `$.parseQuery.separator = ';'`, or, to split on both, `$.parseQuery.separator
= /[&;]/`.


Known Limitations
-----------------

There is no inbuilt support for constructing nested objects from query strings, as used by frameworks like Rails. I suggest that such support is implemented over the top of $.parseQuery, so the first parse deals with type-coercion and multi-valued parameters, and the second construct objects.

Depending on the exact format used by your framework, the amount you care about precise formatting, and the way you want to handle key collisions, the following may be an acceptable base for such an implementation.

    $.parseObjectQuery = function (options) {
          var params = $.parseQuery(options);
          $.each($.parseQuery(options), function (key, value) {
             var obj = params,
                 parts = key.replace(/\]/g, '').replace(/\[$/, '').split('['),
                 last = parts.pop();
             delete params[key];
             $.each(parts, function (i, part) {
                 obj = obj[part] = obj[part] ? obj[part] : {};
             });
             obj[last] = value;
          });
          return params;
    }

    $.parseObjectQuery("a[b]=c&d=e") == {'a':{'b':'c'}, 'd':'e'}
    $.parseObjectQuery({
        query: "a[b]=c&a[b]=d&a[e]=f",
        array_keys: /a\[b\]$/
    }) == {'a': {'b':['c','d'],'e':'f'}}
