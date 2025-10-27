// 1) Figure out which CatItem_IDs are on-screen to evaluate (module or category view)
ClearCollect(
    colCIUniverse,
    // Preferred: items you’re actually editing in the UI
    If(
        !IsEmpty(colModuleCatItems_Working),                     // module edit path
        ShowColumns(colModuleCatItems_Working, "CatItem_ID"),
        If(                                                     // category-only path (rename if your working table differs)
            !IsEmpty(colCategoryItems_Working),
            ShowColumns(colCategoryItems_Working, "CatItem_ID"),
            // Fallback: pull straight from the list by the selected Category
            ShowColumns(
                If(
                    !IsBlank(drpCategory.Selected.Value),
                    Filter(Skill_Matrix_CategoryItems, Category = drpCategory.Selected.Value),
                    Skill_Matrix_CategoryItems
                ),
                "CatItem_ID"
            )
        )
    )
);

// 2) Normalize to a simple list of IDs
ClearCollect(
    colCIIds,
    Distinct(colCIUniverse, CatItem_ID)   // NOTE: if your Distinct returns .Value, use that in the code below
);

// 3) Snapshot BEFORE values for just those IDs
ClearCollect(
    colCIBefore,
    ForAll(
        colCIIds As t,
        {
            CatItem_ID: Coalesce(t.Result, t.Value),
            Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = Coalesce(t.Result, t.Value), Skill_Type)
        }
    )
);




// AFTER snapshot
ClearCollect(
    colCIAfter,
    ForAll(
        colCIIds As t,
        {
            CatItem_ID: Coalesce(t.Result, t.Value),
            Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = Coalesce(t.Result, t.Value), Skill_Type)
        }
    )
);

// Which CatItems actually changed Skill_Type?
Clear(colCIChanged);
ForAll(
    colCIAfter As a,
    If(
        LookUp(colCIBefore, CatItem_ID = a.CatItem_ID, Skill_Type) <> a.Skill_Type,
        Collect(colCIChanged, { CatItem_ID: a.CatItem_ID })
    )
);

// Deduplicate for safety
ClearCollect(colCIChanged, Distinct(colCIChanged, CatItem_ID));





If(
    CountRows(colCIChanged) > 0,
    With(
        {
            code: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
            req:  Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: code,
                Operation: { Value: "CatItem_Update" },
                Status:    { Value: "Pending" },

                // If your Distinct returned .Value, Text(Result) -> Text(Value)
                Updated_CatItem_IDs: Concat(colCIChanged, Text(Result) & ";"),
                Updated_Count:       CountRows(colCIChanged),

                RequestedAt: Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
                    DisplayName: Coalesce(req.displayName, User().FullName),
                    Email: Coalesce(req.mail, User().Email),
                    Department: Coalesce(req.department, ""),
                    JobTitle: Coalesce(req.jobTitle, ""),
                    Picture: ""
                }
            }
        )
    )
);





Clear(colCIUniverse);
Clear(colCIIds);
Clear(colCIBefore);
Clear(colCIAfter);
