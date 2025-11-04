If(
    IsBlank(varSelectedModuleId),
    /* MODULE-LESS: skill edits only */
    If(
        varUpdCount > 0,
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                Operation: { Value: "ModuleUpdated" },
                Status:   { Value: "Pending" },
                Module_ID: Blank(),              // <= IMPORTANT: blank means "global"
                Module_Name: Blank(),
                Added_CatItem_IDs: "",
                Removed_CatItem_IDs: "",
                Updated_CI_JSON: JSON(colSkillEdits, JSONFormat.Compact),
                Updated_Count: varUpdCount,
                RequestedAt: Now(),
                RequestedBy: {
                  '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                  Claims: "i:0#.f|membership|" & Lower(User().Email),
                  DisplayName: User().FullName,
                  Email: User().Email
                }
            }
        )
    ),
    /* WITH MODULE: your existing patch that includes Added/Removed + Updated_CI_JSON */
    Patch(
        Skill_Matrix_Assignments,
        Defaults(Skill_Matrix_Assignments),
        {
            Title: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
            Operation: { Value: "ModuleUpdated" },
            Status:   { Value: "Pending" },
            Module_ID: varSelectedModuleId,
            Module_Name: varSelectedModuleName,
            Added_CatItem_IDs: Concat(colAddIds, CatItem_ID, ";"),
            Removed_CatItem_IDs: Concat(colRemoveIds, CatItem_ID, ";"),
            Updated_CI_JSON: If(varUpdCount>0, JSON(colSkillEdits, JSONFormat.Compact), ""),
            Added_Count:   varAddCount,
            Removed_Count: varRemoveCount,
            Updated_Count: varUpdCount,
            RequestedAt: Now(),
            RequestedBy: {
              '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
              Claims: "i:0#.f|membership|" & Lower(User().Email),
              DisplayName: User().FullName,
              Email: User().Email
            }
        }
    )
);






If(
    IsBlank(varSelectedModuleId),
    Notify("Queued Skill Type updates: " & varUpdCount & " item(s) across all modules.", NotificationType.Success),
    Notify(
        "Queued for module '" & varSelectedModuleName & "': Added " & varAddCount &
        ", Removed " & varRemoveCount & ", Skill updates " & varUpdCount & ".",
        NotificationType.Success
    )
);



