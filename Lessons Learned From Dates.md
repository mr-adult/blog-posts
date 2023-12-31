# Lessons Learned From Dates
///2023-10-24 This blog describes my lessons learned from becoming my team's date and time subject matter expert and how you can avoid making the same mistakes I did. 

I have been in the field of software engineering for just under three years at this point and in that time I have become my team's subject matter expers when it comes to dates. I have dealt with and tackled many problems regarding dates and issues due to how they were implemented in the first place. I would like to share some of the lessons I've learned with you so you do not have to go through the same pain that I did. None of this is novel advice that cannot be found elsewhere, but it is more condensed than other sources I've seen.

## ISO 8601 is your friend.
If you're not already familiar with the [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) specification, there are a few key takeaways. The big one is that you should **always** send dates between computer systems in the format "yyyy-MM-ddThh:mm:ss\[.mmm\] \[time zone designator\]." As an example, my current timestamp (10/23/2023 at 9:13:05 PM) would be written as 2023-10-23T21:13:05 -5:00. This format is super handy. Because it is part of the ISO standard, basically every date or time library or implementation supports and can parse these dates without issues.

I have spent the last few years running a tech stack of SQL Server for the database, C# ASP.NET for the web server, and TypeScript/JavaScript on the frontend. My team had to write custom logic for handling the transfer of dates between C# and JavaScript when we were using the previous developer's method of sending everything in the U.S. locale time of "MM/dd/yyyy." This worked but was more custom code for us to maintain. Once I switched the code to using the ISO 8601 format, I could delete that custom code because both C# and JavaScript have native implementations of this standard in their DateTime (C#) and Date (JavaScript) object types.

ISO 8601 has one other big benefit: a string sort of ISO 8601 formatted results yields a the list of values in chronological or reverse chronological order depending on the sort order. 

All of the following languages have ISO 8601 support:
- C#
- Java
- JavaScript (+TypeScript by extension)
- Python's datetime module
- SQL (all flavors I know of)
- Rust's chrono and time crates
- And many more...

## Machines use UTC, UIs Use the Current User's Locale
Almost all other date and time-related issues come from using non-standardized time zones in your date logic. My recommendation here is that any UI component should convert the UTC (Universal Time Coordinated) value into the user's locale format at the last second before generating the user-displayed string (typically in the rendering stages of the code). User inputs should also be immediately converted back to UTC and all business logic should be run on UTC values. This ensures consistency and prevents several issues that come from comparing dates in different time zones without realizing it. If your productions server only works with one timezone you're still not off the hook here because daylight savings time is a different timezone than standard time!

## With Dates, Implicit State == Bugs
The last big lesson I've learned is that with dates, any and all implicit state will inevitably end up a bug. It's not a matter of if, but when. 

As a prime example, my team built some integration with another app at the company I work for. There was a problem though: In my team's app, we define a date range as going from midnight on the morning of the start date until midnight on the day after the end date. This means a date range of 1/1/2023-12/31/2023 is actually all dates in the interval 1/1/2023 at midnight (inclusive) to 1/1/2024 at midnight (exclusive). The other team had a different definition. Their end dates were exclusive, meaning the same range would generate an interval of 1/1/2023 at midnight (inclusive) to 12/31/2023 at at midnight (exclusive). The developer who built this integration decided to use implicit state. The location in the code base was the determining factor of which type of range we were looking at. Inevitably, this caused issues when others added dates to objects in our page structure that this translation code didn't know about. 

If I could go back and review his PR I would have told him to make that state explicit and with that explicitly part of the range. We could have avoided several bugs with this approach. The big takeaway here is that you should never use implicit state to carry date information. **Always** make date and time state explicit or you **will** run into bugs.