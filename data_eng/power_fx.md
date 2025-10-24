// === Queue one Assignment when the module actually changed ===
If(
    CountRows(colDidAdd) + CountRows(colDidRemove) + CountRows(colDidUpdate) > 0,

    With(
        {
            code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
            req: Office365Users.UserProfileV2(User().Email),
            moduleId: varSelectedModuleId,         // or existingId
            moduleName: varSelectedModuleName      // or existingName
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: code,
                Operation: { Value: "ModuleUpdated" },   // Choice
                Status:    { Value: "Pending" },         // Choice

                // identify the module
                Module_ID:   moduleId,
                Module_Name: moduleName,

                // change metrics
                Added_Count:   CountRows(colDidAdd),
                Removed_Count: CountRows(colDidRemove),
                Updated_Count: CountRows(colDidUpdate),

                // serialize lists for Flow to iterate (IDs are clean text)
                Added_CatItem_IDs:   Concat(colDidAdd, CatItem_ID, ";"),
                Removed_CatItem_IDs: Concat(colDidRemove, CatItem_ID, ";"),

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
            )
        )
    )
);
