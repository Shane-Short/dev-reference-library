
// Compute counts + operation name
Set(addCnt, CountRows(colAddedRefs));
Set(remCnt, CountRows(colRemovedRefs));
Set(
    opValue,
    If(
        addCnt > 0 && remCnt > 0, "ModCatItemAddRem",
        addCnt > 0,               "ModCatItemAdded",
        remCnt > 0,               "ModCatItemRemoved",
                                   "ModuleUpdated"       // no changes (still useful for audit)
    )
);

// Resolve a text Module_ID for assignment (prefer snapshots)
Set(
    varModuleIdSafe,
    Coalesce(
        If(CountRows(colRefAfterSave) > 0, "" & First(colRefAfterSave).Mod_ID),
        If(CountRows(colRefBeforeSave) > 0, "" & First(colRefBeforeSave).Mod_ID),
        If(!IsBlank(varSelectedModuleId),   "" & varSelectedModuleId, "")
    )
);

// Queue the assignment (write BOTH new + legacy payloads for now)
With(
    {
        code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
        req:  Office365Users.UserProfileV2(User().Email)
    },
    Patch(
        Skill_Matrix_Assignments,
        Defaults(Skill_Matrix_Assignments),
        {
            Title:       code,
            Operation:   { Value: opValue },
            Status:      { Value: "Pending" },

            Module_ID:   varModuleIdSafe,
            Module_Name: varSelectedModuleName,

            // New payloads (Reference_IDs)
            Added_Reference_IDs:   Concat(colAddedRefs,   Reference_ID & ";"),
            Removed_Reference_IDs: Concat(colRemovedRefs, Reference_ID & ";"),

            // Legacy payloads (CatItem_IDs) — keep until Flow updated
            Added_CatItem_IDs:     Concat(colAddedRefs,   CatItem_ID & ";"),
            Removed_CatItem_IDs:   Concat(colRemovedRefs, CatItem_ID & ";"),

            Added_Count:   addCnt,
            Removed_Count: remCnt,

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
);
