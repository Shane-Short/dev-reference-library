// ======================= Quick-Add Module =======================
With(
    { typedName: Trim(txtNewModuleName.Text) },
    If(
        IsBlank(typedName),
        Notify("Type a module name first.", NotificationType.Warning),
        With(
            {
                existing: LookUp(Skill_Matrix_Modules, Title = typedName)
            },
            If(
                !IsBlank(existing),
                // Module already exists — select it and move on
                Set(varSelectedModuleId, "" & existing.Mod_ID);
                Set(varSelectedModuleName, existing.Title);
                Notify("Module already exists — selected.", NotificationType.Information),
                // Create brand-new module, then select it
                With(
                    { newId: GUID() },
                    Patch(
                        Skill_Matrix_Modules,
                        Defaults(Skill_Matrix_Modules),
                        {
                            Mod_ID:      newId,
                            Title:       typedName,
                            IsActive:    true,
                            Created_By:  User().FullName,
                            Created_At:  Now()
                        }
                    );
                    Set(varSelectedModuleId, "" & newId);
                    Set(varSelectedModuleName, typedName);
                    Notify("Module created and selected.", NotificationType.Success)
                )
            )
        )
    )
);

// (Optional) Clear the textbox
Reset(txtNewModuleName);

// (Optional) refresh any module pickers/collections if you cache them
// Refresh(Skill_Matrix_Modules);
// ClearCollect(colAllModules, /* your module source again */)








// ======================= Save Module Config (queue-only; cross-category) =======================

// 0) Guards + user cache
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

If(
    IsBlank(varSelectedModuleId),
    Notify("Pick (or create) a module first.", NotificationType.Warning),
    // ELSE:
    With(
        // 1) Build normalized active-ref ids for the selected module (authoritative)
        {
            activeIds:
                AddColumns(
                    ShowColumns(
                        Filter(
                            Skill_Matrix_Reference,
                            Mod_ID = varSelectedModuleId && IsActive = true
                        ),
                        CatItem_ID
                    ),
                    id, Text(CatItem_ID)
                )
        },

        // 2) From the toggle log, compute Adds (Desired=true but not currently active)
        ClearCollect(
            colAddIds,
            ShowColumns(
                Filter(
                    colToggleLog As t,
                    t.Desired = true &&
                    CountIf(activeIds, id = Text(t.CatItem_ID)) = 0
                ),
                "CatItem_ID"
            )
        );

        // 3) From the toggle log, compute Removes (Desired=false but currently active)
        ClearCollect(
            colRemoveIds,
            ShowColumns(
                Filter(
                    colToggleLog As t,
                    t.Desired = false &&
                    CountIf(activeIds, id = Text(t.CatItem_ID)) > 0
                ),
                "CatItem_ID"
            )
        );

        // 4) (Optional) capture Skill_Type edits from the gallery, if you have a dropdown per row
        //    If you don't have skill edits, leave this section as-is (it will just be empty).
        ClearCollect(
            colTypeEdits,
            Filter(
                AddColumns(
                    galCatItems.AllItems,
                    NewType, Coalesce(drpCIType.Selected.Value, Skill_Type),
                    OldType, Skill_Type
                ),
                NewType <> OldType
            )
        );

        // 5) Counts
        Set(varAddCount,     CountRows(colAddIds));
        Set(varRemoveCount,  CountRows(colRemoveIds));
        Set(varUpdCount,     CountRows(colTypeEdits));
        Set(varCodeNow,      "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

        // 6) If there’s any work, queue a single ModCatItemUpdated assignment row (CatItem_ID lists)
        If(
            varAddCount + varRemoveCount > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: varCodeNow,
                    Operation: { Value: "ModCatItemUpdated" },
                    Status: { Value: "Pending" },

                    Module_ID:   varSelectedModuleId,
                    Module_Name: varSelectedModuleName,

                    Added_CatItem_IDs:   Concat(colAddIds,    CatItem_ID, ";"),
                    Removed_CatItem_IDs: Concat(colRemoveIds, CatItem_ID, ";"),

                    Added_Count:   varAddCount,
                    Removed_Count: varRemoveCount,

                    RequestedAt: Now()
                    // If you have a Person column "RequestedBy", you can add it later.
                }
            )
        );

        // 7) If any skill type edits, queue a CatItem_Updated assignment with IDs
        If(
            varUpdCount > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: { Value: "CatItem_Updated" },
                    Status: { Value: "Pending" },

                    Module_ID:   varSelectedModuleId,
                    Module_Name: varSelectedModuleName,

                    Updated_CatItem_IDs: Concat(colTypeEdits, CatItem_ID, ";"),
                    Updated_Count:       varUpdCount,

                    RequestedAt: Now()
                }
            )
        );

        // 8) Notify and clear the local toggle log
        If(
            (varAddCount + varRemoveCount + varUpdCount) > 0,
            Notify(
                "Queued. Added: " & varAddCount &
                " | Removed: " & varRemoveCount &
                " | Skill-type updates: " & varUpdCount,
                NotificationType.Success
            ),
            Notify("No changes to save.", NotificationType.Information)
        );

        // 9) Clear toggle log so fresh changes can be staged
        Clear(colToggleLog)
    )
);
