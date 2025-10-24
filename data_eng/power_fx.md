// Build "Last saved" text in Pacific time (no 2-arg TimeZoneOffset needed)
Set(
    varLastSavedText,
    With(
        {
            // 1) Get UTC "now"
            nowUtc: DateAdd(Now(), -TimeZoneOffset(Now()), TimeUnit.Minutes),

            // 2) Compute current year's DST window for Pacific (approx by date; accurate for normal use)
            y: Year(Now()),
            mar1: Date(y, 3, 1),
            mar1Dow: Weekday(mar1, StartOfWeek.Sunday),
            firstSunMar: DateAdd(mar1, Mod(7 - mar1Dow, 7), TimeUnit.Days),
            secondSunMar: DateAdd(firstSunMar, 7, TimeUnit.Days), // DST starts 2nd Sunday Mar

            nov1: Date(y, 11, 1),
            nov1Dow: Weekday(nov1, StartOfWeek.Sunday),
            firstSunNov: DateAdd(nov1, Mod(7 - nov1Dow, 7), TimeUnit.Days), // DST ends 1st Sunday Nov

            // 3) Is "now" within PDT window? (date-level check; near-changeover hours may differ)
            inDST: nowUtc >= Date(secondSunMar) && nowUtc < Date(firstSunNov),

            // 4) Pacific offset from UTC: -7h during DST, otherwise -8h
            ptOffsetMinutes: If(inDST, -420, -480),

            // 5) Convert UTC → Pacific
            ptNow: DateAdd(nowUtc, ptOffsetMinutes, TimeUnit.Minutes)
        },
        "Last saved: " & Text(ptNow, "[$-en-US]mmm d, yyyy h:mm tt")
    )
);
