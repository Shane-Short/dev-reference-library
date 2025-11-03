// ======================= Save Module Config (robust + toggle-merge + assignments) =======================

// 0) Who + guards
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

If(
    IsBlank(varSelectedModuleId) || IsBlank(cmbSelectCategory.Selected),
    Notify("Pick a module and a category first.", NotificationType.Warning),
    
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },

        // 1) Snapshot: current ACTIVE refs for this module (just CatItem_ID)
        ClearCollect(
            colRefActiveIds,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                "CatItem_ID"
            )
        );

        // 2) Category’s CatItems (normalized id + CORRECT curSelected)
        ClearCollect(
            colCatBase,
            AddColumns(
                Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                id, Text(CatItem_ID),
                curSelected,
                    CountIf(
                        colRefActiveIds,
                        Text(CatItem_ID) = Text(ThisRecord.CatItem_ID)  // <-- key fix
                    ) > 0
            )
        );

        // 3) Desired = overlay local toggle log on top of curSelected
        ClearCollect(
            colDesired,
            AddColumns(
                colCatBase,
                Desired,
                    With(
                        {
                            tgl: LookUp(
                                    colToggleLog,
                                    CatItem_ID = Text(ThisRecord.CatItem_ID)   // compare as text
                                 )
                        },
                        If(IsBlank(tgl), curSelected, tgl.Desired)
                    )
            )
        );

        // 4) Diffs (adds/removes)
        ClearCollect(
            colAdds,
            Filter(colDesired, Desired = true  && curSelected = false)
        );
        ClearCollect(
            colRemoves,
            Filter(colDesired, Desired = false && curSelected = true)
        );

        // 5) APPLY ADDS: reactivate if exists, else create
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
                                SkillLevel: a.Skill_Type,   // (display name Skill_Type; internal may be SkillLevel)
                                IsActive: true,
                                Created_By: reqUserEmail,
                                Created_At: Now()
                            }
                        )
                    )
                )
            )
        );

        // 6) APPLY REMOVES: soft delete
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

        // 7) (Optional) Skill_Type edits within the category (CategoryItems only)
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

        // 8) Counts (for Notify and Assignments)
        Set(varAddCount,     CountRows(colAdds));
        Set(varRemoveCount,  CountRows(colRemoves));
        Set(varUpdCount,     CountRows(colCI_Changed));

        // 9) Authoritative refresh before building RefID lists for Assignments
        Refresh(Skill_Matrix_Reference);

        // 9.1 Build Added/Removed RefID lists AFTER patches landed
        Set(
            addedRefsText,
            Concat(
                Filter(
                    Skill_Matrix_Reference,
                    Mod_ID = varSelectedModuleId &&
                    IsActive = true &&
                    CountIf(colAdds, Text(CatItem_ID) = Text(ThisRecord.CatItem_ID)) > 0
                ),
                Reference_ID,
                ";"
            )
        );
        Set(
            removedRefsText,
            Concat(
                Filter(
                    Skill_Matrix_Reference,
                    Mod_ID = varSelectedModuleId &&
                    IsActive = false &&
                    CountIf(colRemoves, Text(CatItem_ID) = Text(ThisRecord.CatItem_ID)) > 0
                ),
                Reference_ID,
                ";"
            )
        );

        // 9.2 Write Module delta Assignment
        If(
            varAddCount + varRemoveCount > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: {
                        Value: If(
                            varAddCount > 0 && varRemoveCount > 0,
                            "ModCatItemAddRem",
                            If(varAddCount > 0, "ModCatItemAdded", "ModCatItemRemoved")
                        )
                    },
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
        );

        // 9.3 Write CatItem_Updated Assignment
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

        // 10) Notify
        If(
            (varAddCount + varRemoveCount + varUpdCount) > 0,
            Notify(
                "Saved. Added: " & varAddCount & " | Removed: " & varRemoveCount & " | Skill-type updates: " & varUpdCount,
                NotificationType.Success
            ),
            Notify("No changes to save.", NotificationType.Information)
        );

        // 11) HARD rebuild so the gallery reflects server truth & clear toggle log
        Clear(colToggleLog);

        ClearCollect(
            colRefActiveIds,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                "CatItem_ID"
            )
        );

        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                AddColumns(
                    Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                    id, Text(CatItem_ID)
                ),
                IsSelectedForModule,
                CountIf(colRefActiveIds, Text(CatItem_ID) = id) > 0
            )
        )
    )
);
