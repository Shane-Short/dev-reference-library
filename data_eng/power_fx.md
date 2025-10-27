// Track Skill_Type changes to queue CatItem_Update assignment
Clear(colCIChanged);

ForAll(
    colSkillTypeEdits As e,
    Collect(colCIChanged, { CatItem_ID: e.CatItem_ID })
);

// Deduplicate just in case
ClearCollect(colCIChanged, Distinct(colCIChanged, CatItem_ID));

// If any Skill_Type changes occurred, queue a CatItem_Update assignment
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
                Status: { Value: "Pending" },
                Updated_CatItem_IDs: Concat(colCIChanged, CatItem_ID, ";"),
                Updated_Count: CountRows(colCIChanged),
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
