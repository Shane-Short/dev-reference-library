// Track CatItems whose Skill_Type changed (distinct ids)
Clear(colCIChanged);

ForAll(
    colModuleCatItems_Working As w,
    With(
        { ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = w.CatItem_ID) },
        If(
            !IsBlank(ci) && ci.Skill_Type <> w.Skill_Type,
            // Persist the change if not already done elsewhere
            Patch(
                Skill_Matrix_CategoryItems,
                ci,
                { Skill_Type: w.Skill_Type }
            );
            // Record the changed CatItem
            Collect(colCIChanged, { CatItem_ID: w.CatItem_ID })
        )
    )
);

// De-dup
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
                Title:    code,
                Operation:{ Value: "CatItem_Update" },
                Status:   { Value: "Pending" },

                // Payload for Flow
                Updated_CatItem_IDs: Concat(colCIChanged, CatItem_ID, ";"),
                Updated_Count:       CountRows(colCIChanged),

                RequestedAt: Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
                    DisplayName: Coalesce(req.displayName, User().FullName),
                    Email:       Coalesce(req.mail, User().Email),
                    Department:  Coalesce(req.department, ""),
                    JobTitle:    Coalesce(req.jobTitle, ""),
                    Picture:     ""
                }
            }
        )
    )
);

// optional tidy-up
Clear(colCIChanged);



