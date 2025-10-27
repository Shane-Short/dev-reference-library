// AFTER snapshot of Skill_Type per CatItem_ID
ClearCollect(
    colCIAfter,
    ForAll(
        colCIIds As t,
        {
            CatItem_ID: t.CatItem_ID,
            Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = t.CatItem_ID, Skill_Type)
        }
    )
);

// Diff: which CatItems changed Skill_Type
Clear(colCIChanged);
ForAll(
    colCIAfter As a,
    If(
        LookUp(colCIBefore, CatItem_ID = a.CatItem_ID, Skill_Type) <> a.Skill_Type,
        Collect(colCIChanged, { CatItem_ID: a.CatItem_ID })
    )
);
// De-dupe
ClearCollect(colCIChanged, Distinct(colCIChanged, CatItem_ID));

// Queue EditCI only if something changed
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
                Title:               code,
                Operation:           { Value: "CatItem_Update" },
                Status:              { Value: "Pending" },
                Updated_CatItem_IDs: Concat(colCIChanged, Text(CatItem_ID) & ";"),
                Updated_Count:       CountRows(colCIChanged),
                RequestedAt:         Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims:      "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
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

// (optional) tidy
// Refresh(Skill_Matrix_Assignments);
// Clear(colCIUniverse); Clear(colCIIds); Clear(colCIBefore); Clear(colCIAfter); Clear(colCIChanged);
