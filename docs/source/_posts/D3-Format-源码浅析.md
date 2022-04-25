---
title: D3-Format-源码浅析
date: 2022-04-02 11:55:23
categories: [D3]
tags: [D3]
excerpt: 本文浅析了D3-Format
---

考虑到 JavaScript 有时候不会以我们期望的方式显示数字，我们需要一些格式化工具，比如 d3-format

D3-Format 官网例子:

>    d3.format(".0%")(0.123);  // rounded percentage, "12%"
>    d3.format("($.2f")(-3.5); // localized fixed-point currency, "(£3.50)"
>    d3.format("+20")(42);     // space-filled and signed, "                 +42"
>    d3.format(".^20")(42);    // dot-filled and centered, ".........42........."
>    d3.format(".2s")(42e6);   // SI-prefix with two significant digits, "42M"
>    d3.format("#x")(48879);   // prefixed lowercase hexadecimal, "0xbeef"
>    d3.format(",.2r")(4223);  // grouped thousands with two significant digits, "4,200"

## formatSpecifier

d3-format 采用了和 python3's format 一样的 [specification mini-language](https://docs.python.org/3/library/string.html#format-specification-mini-language)

不过 python3 的 Format Specification Mini-Language 的 语法格式为:

> \[\[fill\]align\]\[sign\]\[#\]\[0\]\[width\]\[grouping_option\]\[.precision\]\[type\]

而 D3 对其做出了一些改动：

> \[\[fill\]align\]\[sign\]\[symbol\]\[0\]\[width\]\[,\]\[.precision\]\[~\]\[type\]

在 sign 中 增加了 \\ 和 \( 两个字符

没有 #, 取而代之的是 symbol，等价于 增加了 $ 符号

没有 grouping_option， 取而代之的是 , 符号， 等价于 去除了 _ 符号

增加了一个 ~ 符号

完整 正则表达式 如下：

>    ^\(?:\(.\)?\(\[<>=^\]\)\)?\(\[+\\-\( \]\)?(\[$#\]\)?\(0\)?\(\\d+\)?\(,\)?\(\\.\d+\)?\(~\)?\(\[a-z%\]\)?$

注：i 代表不区分大小写

D3 中 FormatSpecifier 负责解析所需格式

{% spoiler "formatSpecifier 实现相关源码" %}
```javascript
// [[fill]align][sign][symbol][0][width][,][.precision][~][type]
var re = /^(?:(.)?([<>=^]))?([+\-( ])?([$#])?(0)?(\d+)?(,)?(\.\d+)?(~)?([a-z%])?$/i;

export default function formatSpecifier(specifier) {
  if (!(match = re.exec(specifier))) throw new Error("invalid format: " + specifier);
  var match;
  return new FormatSpecifier({
    fill: match[1],
    align: match[2],
    sign: match[3],
    symbol: match[4],
    zero: match[5],
    width: match[6],
    comma: match[7],
    precision: match[8] && match[8].slice(1),
    trim: match[9],
    type: match[10]
  });
}

formatSpecifier.prototype = FormatSpecifier.prototype; // instanceof

export function FormatSpecifier(specifier) {
  this.fill = specifier.fill === undefined ? " " : specifier.fill + "";
  this.align = specifier.align === undefined ? ">" : specifier.align + "";
  this.sign = specifier.sign === undefined ? "-" : specifier.sign + "";
  this.symbol = specifier.symbol === undefined ? "" : specifier.symbol + "";
  this.zero = !!specifier.zero;
  this.width = specifier.width === undefined ? undefined : +specifier.width;
  this.comma = !!specifier.comma;
  this.precision = specifier.precision === undefined ? undefined : +specifier.precision;
  this.trim = !!specifier.trim;
  this.type = specifier.type === undefined ? "" : specifier.type + "";
}

FormatSpecifier.prototype.toString = function() {
  return this.fill
      + this.align
      + this.sign
      + this.symbol
      + (this.zero ? "0" : "")
      + (this.width === undefined ? "" : Math.max(1, this.width | 0))
      + (this.comma ? "," : "")
      + (this.precision === undefined ? "" : "." + Math.max(0, this.precision | 0))
      + (this.trim ? "~" : "")
      + this.type;
};
```
{% endspoiler %}

## formatTypes

在知道了类型以后，由 formatTypes 进行类型处理

{% spoiler "formatTypes 实现相关源码" %}
```javascript
import formatDecimal from "./formatDecimal.js";
import formatPrefixAuto from "./formatPrefixAuto.js";
import formatRounded from "./formatRounded.js";

export default {
  "%": (x, p) => (x * 100).toFixed(p),
  "b": (x) => Math.round(x).toString(2),
  "c": (x) => x + "",
  "d": formatDecimal,
  "e": (x, p) => x.toExponential(p),
  "f": (x, p) => x.toFixed(p),
  "g": (x, p) => x.toPrecision(p),
  "o": (x) => Math.round(x).toString(8),
  "p": (x, p) => formatRounded(x * 100, p),
  "r": formatRounded,
  "s": formatPrefixAuto,
  "X": (x) => Math.round(x).toString(16).toUpperCase(),
  "x": (x) => Math.round(x).toString(16)
};

```
{% endspoiler %}

## locale 

local 采用了默认导出的方式 导出了一个默认函数， 该函数是一个闭包，返回 函数newFormat 和 函数formatPrefix

{% spoiler "locale 实现相关源码" %}
```javascript
import exponent from "./exponent.js";
import formatGroup from "./formatGroup.js";
import formatNumerals from "./formatNumerals.js";
import formatSpecifier from "./formatSpecifier.js";
import formatTrim from "./formatTrim.js";
import formatTypes from "./formatTypes.js";
import {prefixExponent} from "./formatPrefixAuto.js";
import identity from "./identity.js";

var map = Array.prototype.map,
    prefixes = ["y","z","a","f","p","n","µ","m","","k","M","G","T","P","E","Z","Y"];

export default function(locale) {
  var group = locale.grouping === undefined || locale.thousands === undefined ? identity : formatGroup(map.call(locale.grouping, Number), locale.thousands + ""),
      currencyPrefix = locale.currency === undefined ? "" : locale.currency[0] + "",
      currencySuffix = locale.currency === undefined ? "" : locale.currency[1] + "",
      decimal = locale.decimal === undefined ? "." : locale.decimal + "",
      numerals = locale.numerals === undefined ? identity : formatNumerals(map.call(locale.numerals, String)),
      percent = locale.percent === undefined ? "%" : locale.percent + "",
      minus = locale.minus === undefined ? "−" : locale.minus + "",
      nan = locale.nan === undefined ? "NaN" : locale.nan + "";

  function newFormat(specifier) {...}

  function formatPrefix(specifier, value) {
    var f = newFormat((specifier = formatSpecifier(specifier), specifier.type = "f", specifier)),
        e = Math.max(-8, Math.min(8, Math.floor(exponent(value) / 3))) * 3,
        k = Math.pow(10, -e),
        prefix = prefixes[8 + e / 3];
    return function(value) {
      return f(k * value) + prefix;
    };
  }

  return {
    format: newFormat,
    formatPrefix: formatPrefix
  };
}

```
{% endspoiler %}

函数newFormat 也是个闭包， 对specifier进行处理， 返回了一个函数format

{% spoiler "newFormat 实现相关源码" %}
```javascript
export default function(locale) {
  function newFormat(specifier) {
    specifier = formatSpecifier(specifier);

    var fill = specifier.fill,
        align = specifier.align,
        sign = specifier.sign,
        symbol = specifier.symbol,
        zero = specifier.zero,
        width = specifier.width,
        comma = specifier.comma,
        precision = specifier.precision,
        trim = specifier.trim,
        type = specifier.type;

    // The "n" type is an alias for ",g".
    if (type === "n") comma = true, type = "g";

    // The "" type, and any invalid type, is an alias for ".12~g".
    else if (!formatTypes[type]) precision === undefined && (precision = 12), trim = true, type = "g";

    // If zero fill is specified, padding goes after sign and before digits.
    if (zero || (fill === "0" && align === "=")) zero = true, fill = "0", align = "=";

    // Compute the prefix and suffix.
    // For SI-prefix, the suffix is lazily computed.
    var prefix = symbol === "$" ? currencyPrefix : symbol === "#" && /[boxX]/.test(type) ? "0" + type.toLowerCase() : "",
        suffix = symbol === "$" ? currencySuffix : /[%p]/.test(type) ? percent : "";

    // What format function should we use?
    // Is this an integer type?
    // Can this type generate exponential notation?
    var formatType = formatTypes[type],
        maybeSuffix = /[defgprs%]/.test(type);

    // Set the default precision if not specified,
    // or clamp the specified precision to the supported range.
    // For significant precision, it must be in [1, 21].
    // For fixed precision, it must be in [0, 20].
    precision = precision === undefined ? 6
        : /[gprs]/.test(type) ? Math.max(1, Math.min(21, precision))
        : Math.max(0, Math.min(20, precision));

    function format(value) { ... }

    return format;
  }
  ...
}
```
{% endspoiler %}

在 format 中， 对value进行了格式化操作

{% spoiler "format 实现相关源码" %}
```javascript
export default function(locale) {
  function newFormat(specifier) {
    function format(value) {
      var valuePrefix = prefix,
          valueSuffix = suffix,
          i, n, c;

      if (type === "c") {
        valueSuffix = formatType(value) + valueSuffix;
        value = "";
      } else {
        value = +value;

        // Determine the sign. -0 is not less than 0, but 1 / -0 is!
        var valueNegative = value < 0 || 1 / value < 0;

        // Perform the initial formatting.
        value = isNaN(value) ? nan : formatType(Math.abs(value), precision);

        // Trim insignificant zeros.
        if (trim) value = formatTrim(value);

        // If a negative value rounds to zero after formatting, and no explicit positive sign is requested, hide the sign.
        if (valueNegative && +value === 0 && sign !== "+") valueNegative = false;

        // Compute the prefix and suffix.
        valuePrefix = (valueNegative ? (sign === "(" ? sign : minus) : sign === "-" || sign === "(" ? "" : sign) + valuePrefix;
        valueSuffix = (type === "s" ? prefixes[8 + prefixExponent / 3] : "") + valueSuffix + (valueNegative && sign === "(" ? ")" : "");

        // Break the formatted value into the integer “value” part that can be
        // grouped, and fractional or exponential “suffix” part that is not.
        if (maybeSuffix) {
          i = -1, n = value.length;
          while (++i < n) {
            if (c = value.charCodeAt(i), 48 > c || c > 57) {
              valueSuffix = (c === 46 ? decimal + value.slice(i + 1) : value.slice(i)) + valueSuffix;
              value = value.slice(0, i);
              break;
            }
          }
        }
      }

      // If the fill character is not "0", grouping is applied before padding.
      if (comma && !zero) value = group(value, Infinity);

      // Compute the padding.
      var length = valuePrefix.length + value.length + valueSuffix.length,
          padding = length < width ? new Array(width - length + 1).join(fill) : "";

      // If the fill character is "0", grouping is applied after padding.
      if (comma && zero) value = group(padding + value, padding.length ? width - valueSuffix.length : Infinity), padding = "";

      // Reconstruct the final output based on the desired alignment.
      switch (align) {
        case "<": value = valuePrefix + value + valueSuffix + padding; break;
        case "=": value = valuePrefix + padding + value + valueSuffix; break;
        case "^": value = padding.slice(0, length = padding.length >> 1) + valuePrefix + value + valueSuffix + padding.slice(length); break;
        default: value = padding + valuePrefix + value + valueSuffix; break;
      }

      return numerals(value);
    }

    format.toString = function() {
      return specifier + "";
    };
    ...
  }
}
```
{% endspoiler %}

## defaultLocale

最后 将其暴露在d3空间中

{% spoiler "defaultLocale 实现相关源码" %}
```javascript
import formatLocale from "./locale.js";

var locale;
export var format;
export var formatPrefix;

defaultLocale({
  thousands: ",",
  grouping: [3],
  currency: ["$", ""]
});

export default function defaultLocale(definition) {
  locale = formatLocale(definition);
  format = locale.format;
  formatPrefix = locale.formatPrefix;
  return locale;
}
```
{% endspoiler %}
