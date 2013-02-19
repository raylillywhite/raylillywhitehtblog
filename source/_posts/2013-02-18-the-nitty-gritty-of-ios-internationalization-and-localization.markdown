---
layout: post
title: "The Nitty Gritty of iOS Internationalization and Localization"
date: 2013-02-18 22:30
author: Ray Lillywhite
comments: true
categories: iOS
---

The best way to prevent hating your job when your company grows is to use the Apple provided localization tools from day one, even if you have no plans to localize your app. Maybe it's not the best way, but it's certainly required. From day one you should be using:

* `NSNumberFormatter`
* `NSDateFormatter`
* `NSLocalizedString`
* `[[NSLocale currentLocale] objectForKey:NSLocaleUsesMetricSystem]`

From day one, you should not be:

* Pluralizing by adding an 's' (Safe yourself some trouble and use separate localized strings for plural and singular nouns)
* Using magic numbers to set static sizes for views (specifically views with text)
* Using '$' or other currency symbols in string formats

Let's start with the easier ones.

## NSNumberFormatter

NSNumberFormatter's most important use in localization for us was with displaying currencies. Users will have a different expectation of how a given currency is displayed depending on what country they are from. A price in US Dollars could be shown as `$50.00`, `US$50.00`, `50,00 $US`, etc. 

NSNumberFormatter's `NSNumberFormatterCurrencyStyle` allows you to let Apple decide which one to show the user, which gives them a consistent experience across all of their apps. Showing the full currency code (USD) would also work, and may be useful for certain detailed views like a verbose receipt, but we chose to stick with the more minimal format that NSNumberFormatter provides.
Example:
```objc
NSNumberFormatter *currencyFormatter = [[NSNumberFormatter alloc] init];
[currencyFormatter setNumberStyle:NSNumberFormatterCurrencyStyle];
[currencyFormatter setCurrencyCode:currencyCode];
[currencyFormatter setMaximumFractionDigits:0];
[currencyFormatter setRoundingMode:NSNumberFormatterRoundHalfUp];
return [currencyFormatter stringFromNumber:@(amount)];
```

Even if you aren't displaying prices, make sure to use NSNumberFormatter to ensure that large numbers and decimals are displayed to the user correctly. `,` and `.` are switched in other locales.

### Caching and Thread Safety

NSNumberFormatters and NSDateFormatters are slow to initialize and configure, so you generally want to cache any formatters that you can use in multiple places. Unfortunately they aren't thread-safe though, so we keep separate formatters per thread:

```objc
NSString *HTCurrencyString(double amount, NSString *currencyCode)
{
    static NSString *currencyFormatterKey = @"HTCurrencyFormatter";
    NSNumberFormatter *currencyFormatter = [[NSThread currentThread] threadDictionary][currencyFormatterKey];
    if (currencyFormatter == nil) 
    {
        currencyFormatter = [[NSNumberFormatter alloc] init];
        [currencyFormatter setNumberStyle:NSNumberFormatterCurrencyStyle];
        [currencyFormatter setLocale:[NSLocale currentLocale]];
 
        [[NSThread currentThread] threadDictionary][currencyFormatterKey] = currencyFormatter;
    }
    [currencyFormatter setCurrencyCode:currencyCode];
    [currencyFormatter setMaximumFractionDigits:0];
    [currencyFormatter setRoundingMode:NSNumberFormatterRoundHalfUp];
    return [currencyFormatter stringFromNumber:@(amount)];
}
```

## NSDateFormatter
Using date formatters involves a couple of additional tricks, beyond the caching necessary for both date and number formatters. There are many built-in date formatter styles that we made use of with cached formatters like:

```objc
NSDateFormatter *HTMediumDateFormatter()
{
    static NSString *mediumDateFormatterKey = @"HTMediumDateFormatter";
    NSDateFormatter *dateFormatter = [[NSThread currentThread] threadDictionary][mediumDateFormatterKey];
    if (dateFormatter == nil)
    {
        dateFormatter = [[NSDateFormatter alloc] init];
        [dateFormatter setDateStyle:NSDateFormatterMediumStyle];
        [[NSThread currentThread] threadDictionary][mediumDateFormatterKey] = dateFormatter;
    }
    return dateFormatter;
}
```

But there were also cases that weren't really covered by the built-in styles, such as the date format that we prefer on our checkout screen, "Mon, Jan 28". We previously were showing that by setting the date format with the string "EEE, MMM d". This format doesn't make sense for other countries though. We thought we would have to revert to using one of the built-in styles, but we found a very helpful method that Apple added in iOS 4.0: `+[NSDateFormatter dateFormatFromTemplate:options:locale:]`. That method will take the components of a date that you want, such as an abbreviated weekday, an abbreviated month and a day, and create a format string that suites the given locale.

```objc
        [_dateFormatter setDateFormat:[NSDateFormatter dateFormatFromTemplate:@"EEEMMMd"
                                                                      options:0
                                                                       locale:[NSLocale currentLocale]]];
```

Depending on the user's locale, that date formatter will give you strings like the following:

* US: `Mon, Jan 28`
* UK: `Mon 28 Jan`
* FR: `lun. 28 janv.`

Way easier and more reliable than trying to manipulate the built-in formats into the one that you're looking for ;)  If you go through your app and replace your hardcoded date formats using this method, be careful not to don't replace any date formats that you're using for parsing.

### Date Formatter Gotcha

When using a date formatter to parse dates from an API, make sure to set the locale to `@"en_US_POSIX"` for consistent behavior across devices/users.

```objc
NSDateFormatter *HTRFC822DateFormatter()
{
    static NSString *rfc822DateFormatterKey = @"HTRFC822DateFormatter";
    NSDateFormatter *dateFormatter = [[NSThread currentThread] threadDictionary][rfc822DateFormatterKey];
    if (dateFormatter == nil)
    {
        dateFormatter = [[NSDateFormatter alloc] init];
        [dateFormatter setLocale:[[NSLocale alloc] initWithLocaleIdentifier:@"en_US_POSIX"]];
        dateFormatter.dateFormat = @"EEE, dd MMM yyyy HH:mm:ss ZZZ";
        [[NSThread currentThread] threadDictionary][rfc822DateFormatterKey] = dateFormatter;
    }
    dateFormatter.timeZone = [NSTimeZone systemTimeZone];
    return dateFormatter;
}
```

[More information on this from Apple](http://developer.apple.com/library/ios/#qa/qa1480/_index.html)


## NSLocalizedString

The most important part of localization: user facing strings. Apple provides NSLocalizedString to wrap any string or format that you'll be displaying to the user. [@Mattt](http://twitter.com/mattt) has a [great article about NSLocalizedString on NSHipster](http://nshipster.com/nslocalizedstring/), so instead of going into detail about it here, I'm just going to describe some things that we did differently, so that you can decide if you'd want to do something similar.

As Mattt pointed out, the most common NSLocalizedString method is `NSString *NSLocalizedString(NSString *key, NSString *comment)`.  The basic behavior of that is to lookup the key in a `.strings` file that matches the preferred language of the user. We chose to use `NSLocalizedStringWithDefaultValue` though, because `NSLocalizedString(key, comment)` leaves you two straight-forward options:

1. Use an English strings file (`en.lproj/Localizable.strings`) to map your keys to english strings. This means that your code no longer shows your english strings in context. That will increase the amount of jumping between files that you need to do, and can cause confusion if the keys look like English strings, but are out of sync with the strings file.

2. Keep your English strings in code as the keys. In this case, keep your english strings file empty, and after you run genstrings to create your English strings file for your translators, discard it. We keep an empty english strings file in our repository with a comment in it that lets developers know that it's intentionally empty, but I'm not sure if having an empty English strings file behaves any differently than just not having an English one. The downside to this approach is that modifying your English strings (which are your keys) will cause your strings in other languages not to line up anymore. In cases where the meaning of the string changed, that makes sense. But in cases where you're fixing typos or rewording things that wouldn't require re-translation, you could be causing extra work for your translators.

I'd definitely recommend the second option, leaving the English strings file blank, because it'll make development less painful. In the past, I've used the first option and it was no fun to regularly switch back and forth between code and the strings file, or to search for a user-facing string by looking up the key in the strings file. The reason we chose to use the more complicated `NSLocalizedStringWithDefaultValue` is to prevent changes to English strings breaking the keys for the strings files as would normally happen by using the empty English strings file approach.

We created a macro (with a much shorter name) to minimize our pain with using NSLocalizedStringWithDefaultValue:

```objc
#define HTStr(key, value, comment) NSLocalizedStringWithDefaultValue(key, nil, [NSBundle mainBundle], value, comment)
```

Example usage:

```objc
self.emailField.text = HTStr(@"S_Email", @"Email", @"Placeholder for login email field")
```

With this we have the option of changing just the English string, to prevent retranslation and breaking links to existing translations. Or, we can change the key and the value to trigger a retranslation if the meaning of the English string has changed at all. There are a couple downsides to this approach though. Using our macro caused genstrings to not find our strings, so we had to hack together a [script for generating our strings file](https://gist.github.com/raylillywhite/4984291). And you could accidentally use the same key for two different values. Fortunately, this is caught by genstrings:

```
Key "S_Email" used with multiple values. Value "Email" kept. Value "E-mail" ignored.
```

In hindsight though, we probably would have been fine with using the simple `NSLocalizedString(key, comment)`. Smartling makes it pretty easy for us to get our strings files updated by the translators, so if keys were to change anytime we change the English text, that wouldn't be a big problem. And you could always selectively use `NSLocalizedStringWithDefaultValue` in that case.