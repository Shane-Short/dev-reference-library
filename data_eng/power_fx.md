// Take a BEFORE snapshot of Reference rows for this module (by Reference_ID)
ClearCollect(
    colRefBeforeSave,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
        "Reference_ID", "CatItem_ID"
    )
);



// Take an AFTER snapshot (post-save)
ClearCollect(
    colRefAfterSave,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
        "Reference_ID", "CatItem_ID"
    )
);

// Compute diffs by Reference_ID
ClearCollect(
    colAddedRefs,
    Filter(
        colRefAfterSave As a,
        CountIf(colRefBeforeSave, Reference_ID = a.Reference_ID) = 0
    )
);

ClearCollect(
    colRemovedRefs,
    Filter(
        colRefBeforeSave As b,
        CountIf(colRefAfterSave, Reference_ID = b.Reference_ID) = 0
    )
);

// Be defensive: ensure we have Module_ID even if the var is blank
Set(
    varModuleIdSafe,
    If(
        !IsBlank(varSelectedModuleId),
        varSelectedModuleId,
        LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID)
    )
);

// Queue the ModuleUpdated assignment with Reference_ID payloads
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
            Operation: { Value: "ModuleUpdated" },
            Status:    { Value: "Pending" },

            // Identify the module
            Module_ID:   varModuleIdSafe,
            Module_Name: varSelectedModuleName,

            // Provide Reference_ID lists for Flow (best key for Entries)
            Added_Reference_IDs:   Concat(colAddedRefs,   Reference_ID, ";"),
            Removed_Reference_IDs: Concat(colRemovedRefs, Reference_ID, ";"),

            // Counts (by Reference_ID)
            Added_Count:   CountRows(colAddedRefs),
            Removed_Count: CountRows(colRemovedRefs),

            // If you still track skill-type edits separately:
            Updated_Count: CountRows(Coalesce(colDidUpdate, Table())),

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

// (Recommended) refresh + clear temp collections
Refresh(Skill_Matrix_Reference);
Refresh(Skill_Matrix_Assignments);
Clear(colRefBeforeSave);
Clear(colRefAfterSave);
Clear(colAddedRefs);
Clear(colRemovedRefs);
