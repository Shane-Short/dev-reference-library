// === Queue ModuleUpdated assignment (diff current working set vs. persisted Reference) ===
If(
    CountRows(colDidAdd) + CountRows(colDidRemove) + CountRows(colDidUpdate) > 0,

    // AFTER = current selection; guarantee CatItem_ID exists per row
    ClearCollect(
        colAfterCatItems,
        ForAll(
            colModuleCatItems_Working As w,
            {
                CatItem_ID: Coalesce(
                    w.CatItem_ID,
                    // fallback via CategoryItems table
                    LookUp(
                        Skill_Matrix_CategoryItems,
                        Category = w.Category && Item = w.Item,
                        CatItem_ID
                    ),
                    // final fallback via Reference (scoped to this module)
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && Category = w.Category && Item = w.Item,
                        CatItem_ID
                    )
                )
            }
        )
    );

    // BEFORE = what Reference currently has for this module
    ClearCollect(
        colBeforeCatItems,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
            "CatItem_ID"
        )
    );

    // Added = in AFTER but not in BEFORE
    ClearCollect(
        colAddedCatItems,
        Filter(
            colAfterCatItems As a,
            !IsBlank(a.CatItem_ID) &&
            CountIf(colBeforeCatItems, CatItem_ID = a.CatItem_ID) = 0
        )
    );

    // Removed = in BEFORE but not in AFTER
    ClearCollect(
        colRemovedCatItems,
        Filter(
            colBeforeCatItems As b,
            !IsBlank(b.CatItem_ID) &&
            CountIf(colAfterCatItems, CatItem_ID = b.CatItem_ID) = 0
        )
    );

    // Create one Assignment row capturing the change
    With(
        {
            code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
            req:  Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: code,
                Operation: { Value: "ModuleUpdated" },   // Choice
                Status:    { Value: "Pending" },         // Choice

                Module_ID:   varSelectedModuleId,
                Module_Name: varSelectedModuleName,

                // Diff payload for Flow
                Added_CatItem_IDs:   Concat(colAddedCatItems,   CatItem_ID, ";"),
                Removed_CatItem_IDs: Concat(colRemovedCatItems, CatItem_ID, ";"),
                Added_Count:         CountRows(colAddedCatItems),
                Removed_Count:       CountRows(colRemovedCatItems),
                Updated_Count:       CountRows(colDidUpdate),

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
