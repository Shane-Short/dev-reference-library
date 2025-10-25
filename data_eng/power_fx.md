// === Queue ModuleUpdated assignment (compute diffs first, then Patch) ===
If(
    CountRows(colDidAdd) + CountRows(colDidRemove) + CountRows(colDidUpdate) > 0,
    With(
        {
            code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
            req: Office365Users.UserProfileV2(User().Email),

            // BEFORE = what Reference currently has for this module
            beforeTbl: Filter(Skill_Matrix_Reference As r, r.Mod_ID = varSelectedModuleId),

            // AFTER = what the working selection currently holds (must include CatItem_ID)
            afterTbl:  colModuleCatItems_Working,

            // Diff tables
            addTbl: Filter(afterTbl As w, CountIf(beforeTbl, CatItem_ID = w.CatItem_ID) = 0),
            remTbl: Filter(beforeTbl As r, CountIf(afterTbl, CatItem_ID = r.CatItem_ID) = 0)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: code,
                Operation: { Value: "ModuleUpdated" },
                Status:    { Value: "Pending" },

                Module_ID:   varSelectedModuleId,
                Module_Name: varSelectedModuleName,

                // Use the precomputed tables here (no nested With/records inside fields)
                Added_CatItem_IDs:   Concat(addTbl, CatItem_ID, ";"),
                Removed_CatItem_IDs: Concat(remTbl, CatItem_ID, ";"),
                Added_Count:         CountRows(addTbl),
                Removed_Count:       CountRows(remTbl),
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
)


// when you detect an ADD
Collect(colDidAdd, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });

// when you detect a REMOVE
Collect(colDidRemove, { Reference_ID: ref.Reference_ID, CatItem_ID: ref.CatItem_ID });
