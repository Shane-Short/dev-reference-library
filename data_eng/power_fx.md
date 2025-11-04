// ======================= Save Module Config (diff from working + toggles) =======================

// 0) Guard & who
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

If(
    IsBlank(varSelectedModuleId) || IsBlank(varSelectedCategoryText),
    Notify("Pick a module and a category first.", NotificationType.Warning),
    
    /* ELSE */ With(
        {
            // We already have the working list with IsSelectedForModule + CatItem_ID
            // colModuleCatItems_Working
            // Toggles are in colToggleLog with { CatItem_ID, Desired }
            desiredTable:
                AddColumns(
                    colModuleCatItems_Working,
                    Desired,
                        With(
                            { t: LookUp(colToggleLog, CatItem_ID = CatItem_ID) },
                            If( IsBlank(t), IsSelectedForModule, t.Desired )
                        )
                )
        },

        // 1) Compute diffs strictly from working + toggles
        ClearCollect(
            colAdds,
            Filter(desiredTable, Desired = true && IsSelectedForModule = false)
        );
        ClearCollect(
            colRemoves,
            Filter(desiredTable, Desired = false && IsSelectedForModule = true)
        );

        // 2) APPLY ADDS (reactivate if exists; else create)
        ForAll(
            colAdds As a,
            With(
                {
                    existingRef: LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && Text(CatItem_ID) = Text(a.CatItem_ID)
                    )
                },
                If(
                    !IsBlank(existingRef),
                    Patch(
                        Skill_Matrix_Reference,
                        existingRef,
                        { IsActive: true, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                    ),
                    With(
                        { newRefId: GUID() },
                        Patch(
                            Skill_Matrix_Reference,
                            Defaults(Skill_Matrix_Reference),
                            {
                                Reference_ID: newRefId,
                                Mod_ID: varSelectedModuleId,
                                Module: varSelectedModuleName,
                                CatItem_ID: a.CatItem_ID,
                                Category: a.Category,
                                Item: a.Item,
                                SkillLevel: a.Skill_Type,
                                IsActive: true,
                                Created_By: reqUserEmail,
                                Created_At: Now()
                            }
                        )
                    )
                )
            )
        );

        // 3) APPLY REMOVES (soft delete)
        ForAll(
            colRemoves As r,
            With(
                {
                    activeRef: LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId &&
                        Text(CatItem_ID) = Text(r.CatItem_ID) &&
                        IsActive = true
                    )
                },
                If(
                    !IsBlank(activeRef),
                    Patch(
                        Skill_Matrix_Reference,
                        activeRef,
                        { IsActive: false, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                    )
                )
            )
        );

        // 4) (Optional) Skill_Type edits inside the category list
        ClearCollect(
            colCI_Changed,
            Filter(
                AddColumns(
                    galCatItems.AllItems,
                    NewType, Coalesce(drpCIType.Selected.Value, Skill_Type),
                    OldType, Skill_Type
                ),
                NewType <> OldType
            )
        );
        ForAll(
            colCI_Changed As c,
            Patch(
                Skill_Matrix_CategoryItems,
                LookUp(Skill_Matrix_CategoryItems, CatItem_ID = c.CatItem_ID),
                { Skill_Type: c.NewType, Modified_By: reqUserEmail, Modified_At: Now() }
            )
        );

        // 5) Counts + Assignments
        Set(varAddCount,     CountRows(colAdds));
        Set(varRemoveCount,  CountRows(colRemoves));
        Set(varUpdCount,     CountRows(colCI_Changed));
        Set(varCodeNow,      "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

        // 5.1 Module delta log (Reference_ID lists)
        If(
            varAddCount + varRemoveCount > 0,
            With(
                {
                    addedRefsText:
                        Concat(
                            ShowColumns(
                                AddColumns(
                                    colAdds,
                                    Reference_ID,
                                        Coalesce(
                                            LookUp(
                                                Skill_Matrix_Reference,
                                                Mod_ID = varSelectedModuleId && Text(CatItem_ID) = Text(CatItem_ID)
                                            ).Reference_ID,
                                            LookUp(
                                                Skill_Matrix_Reference,
                                                Mod_ID = varSelectedModuleId && Text(CatItem_ID) = Text(CatItem_ID) && IsActive = true
                                            ).Reference_ID
                                        )
                                ),
                                "Reference_ID"
                            ),
                            Reference_ID,
                            ";"
                        ),
                    removedRefsText:
                        Concat(
                            ShowColumns(
                                AddColumns(
                                    colRemoves,
                                    Reference_ID,
                                        LookUp(
                                            Skill_Matrix_Reference,
                                            Mod_ID = varSelectedModuleId && Text(CatItem_ID) = Text(CatItem_ID)
                                        ).Reference_ID
                                ),
                                "Reference_ID"
                            ),
                            Reference_ID,
                            ";"
                        )
                },
                Patch(
                    Skill_Matrix_Assignments,
                    Defaults(Skill_Matrix_Assignments),
                    {
                        Title: varCodeNow,
                        Operation: { Value: If(varAddCount>0 && varRemoveCount>0, "ModCatItemAddRem", If(varAddCount>0, "ModCatItemAdded", "ModCatItemRemoved")) },
                        Status: { Value: "Pending" },
                        Module_ID: varSelectedModuleId,
                        Module_Name: varSelectedModuleName,
                        Added_Reference_IDs: addedRefsText,
                        Removed_Reference_IDs: removedRefsText,
                        Added_Count: varAddCount,
                        Removed_Count: varRemoveCount,
                        RequestedAt: Now()
                    }
                )
            )
        );

        // 5.2 CatItem skill-type update log
        If(
            varUpdCount > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: { Value: "CatItem_Updated" },
                    Status: { Value: "Pending" },
                    Module_ID: varSelectedModuleId,
                    Module_Name: varSelectedModuleName,
                    Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
                    Updated_Count: varUpdCount,
                    RequestedAt: Now()
                }
            )
        );

        // 6) Notify
        If(
            varAddCount + varRemoveCount + varUpdCount > 0,
            Notify(
                "Saved. Added: " & varAddCount & " | Removed: " & varRemoveCount & " | Skill-type updates: " & varUpdCount,
                NotificationType.Success
            ),
            Notify("No changes to save.", NotificationType.Information)
        );

        // 7) Refresh and rebuild (authoritative)
        Refresh(Skill_Matrix_Reference);
        Clear(colToggleLog);

        // Rebuild active refs (single-column id)
        ClearCollect(
            colRefActiveIds,
            RenameColumns(
                ShowColumns(
                    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                    "CatItem_ID"
                ),
                "CatItem_ID", "id"
            )
        );

        // Rebuild working set for the current category
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                ShowColumns(
                    Filter(
                        Skill_Matrix_CategoryItems,
                        Text(Category) = varSelectedCategoryText
                    ),
                    "CatItem_ID", "Category", "Item", "Skill_Type"
                ),
                id, Text(CatItem_ID),
                IsSelectedForModule, CountIf(colRefActiveIds, id = Text(CatItem_ID)) > 0
            )
        )
    )
);









