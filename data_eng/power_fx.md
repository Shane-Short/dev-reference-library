// Vars you already use
Set(varMe, Lower(Trim(Coalesce(Office365Users.MyProfileV2().mail, User().Email))));

// Create error collections with schema, then clear
ClearCollect(colEntryErrors, Table({ Error: "", Column: "" })); Clear(colEntryErrors);
ClearCollect(colSaveErrors,  Table({ Module: "", Category: "", Item: "", Message: "" })); Clear(colSaveErrors);

// (keep the rest of your existing OnVisible after this)


// Use a real Reference row so required fields (if any) are satisfied
With(
    { r: First(Skill_Matrix_Reference) },
    Patch(
        Skill_Matrix_Entries,
        Defaults(Skill_Matrix_Entries),
        {
            Title: "_DirectWriteTest_",
            Employee: "Debug User",
            Employee_Email: varMe,
            Module: "_TestModule_",
            Category: "_TestCat_",
            Item: "_TestItem_",
            Skill_Level: 1,
            Reference_ID: r.Reference_ID,
            Mod_ID: r.Mod_ID,
            Timestamp: Now()
        }
    )
);

// Capture server-side errors from this datasource
Clear(colEntryErrors);
Collect(colEntryErrors, Errors(Skill_Matrix_Entries));

// Visual feedback
Notify(
    "Direct write attempted. Errors: " & Text(CountRows(colEntryErrors)),
    If(CountRows(colEntryErrors) > 0, NotificationType.Error, NotificationType.Success)
);

If(
    IsEmpty(colEntryErrors),
    "No Errors",
    Concat(colEntryErrors, Error & " — " & Column, Char(10))
)


