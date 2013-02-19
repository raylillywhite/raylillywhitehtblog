---
layout: post
title: "The Nitty Gritty of iOS Internationalization and Localization"
date: 2013-02-18 22:30
comments: true
categories: 
---

The best way to prevent hating your job when your company grows is to use the Apple provided localization tools from day one, even if you have no plans to localize your app. Maybe it's not the best way, but it's certainly required. From day one you should be using:

* `NSLocalizedString`
* `NSDateFormatter`
* `NSNumberFormatter`
* `[[NSLocale currentLocale] objectForKey:NSLocaleUsesMetricSystem]`

From day one, you should not be:

* Pluralizing by adding an 's' (Safe yourself some trouble and use separate localized strings for plural and singular nouns)
* Using magic numbers to set static sizes for views (specifically views with text)
* Using '$' or other currency symbols in string formats

Let's start with the easier ones.
## NSNumberFormatter
NSNumberFormatter's most important use in localization is with display currencies. Users will have a different expectation of how a given currency is displayed depending on what country they are from. A price in US Dollars could be shown as "$50.00", "US$50.00", "50,00 $US", etc. 

`NSNumberFormatter`'s `NSNumberFormatterCurrencyStyle` allows you to let Apple decide which one to show the user, which gives them a consistent experience across all of their apps. Showing the full currency code (USD) would also work, and may be useful for certain detailed views like a verbose receipt, but we chose to stick with the more minimal format that NSNumberFormatter provides.
Example:
```objc
NSNumberFormatter *currencyFormatter = [[NSNumberFormatter alloc] init];
[currencyFormatter setNumberStyle:NSNumberFormatterCurrencyStyle];
[currencyFormatter setCurrencyCode:currencyCode];
[currencyFormatter setMaximumFractionDigits:0];
[currencyFormatter setRoundingMode:NSNumberFormatterRoundHalfUp];
return [currencyFormatter stringFromNumber:@(amount)];
```

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
https://gist.github.com/6fef4a67115dfdc4e10a
But there were also cases that weren't really covered by the built-in styles, such as the date format that we prefer on our checkout screen, "Mon, Jan 28". We previously were showing that by setting the date format with the string "EEE, MMM d". This format doesn't make sense for other countries though. We thought we would have to revert to using one of the built-in styles, but we found a very helpful method that Apple added in iOS 4.0: `+[NSDateFormatter dateFormatFromTemplate:options:locale:]`. That method will take the components of a date that you want, such as an abbreviated weekday, an abbreviated month and a day, and create a format string that suites the given locale.
```objc
        [_dateFormatter setDateFormat:[NSDateFormatter dateFormatFromTemplate:@"EEEMMMd"
                                                                      options:0
                                                                       locale:[NSLocale currentLocale]]];
```
Depending on the user's locale, that date formatter will give you strings like the following:
* US: "Mon, Jan 28"
* UK: "Mon 28 Jan"
* FR: "lun. 28 janv."

Way easier and more reliable than trying to manipulate the built-in formats into the one that you're looking for ;)

### Date Formatter Gotcha
When using a date formatter to parse dates from an API, you have to be more careful
