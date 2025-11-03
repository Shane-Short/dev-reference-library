// Seed schema for pending toggle log so fields are recognized
Clear(colToggleLog);
Collect(colToggleLog, { CatItem_ID: "", Desired: false });
RemoveIf(colToggleLog, CatItem_ID = "" && Desired = false);

// Usual init (keep whatever you already had, these are safe)
Clear(colModuleCatItems_Working);
Clear(colRefActiveIds);
Set(varSelectedModuleId,   Blank());
Set(varSelectedModuleName, Blank());



// Prefer pending choice in colToggleLog; else use server-truth from working set
If(
    !IsBlank(LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID)),
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID).Desired,
    IsSelectedForModule
)





Patch(
    colToggleLog,
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID),
    { CatItem_ID: ThisItem.CatItem_ID, Desired: true }
)





Patch(
    colToggleLog,
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID),
    { CatItem_ID: ThisItem.CatItem_ID, Desired: false }
)





Set(varSelectedModuleId,   cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// clear pending toggles when switching modules
Clear(colToggleLog);

// (keep your existing rebuild logic for the working set)




// ======================= Save Module Config (robust, keyed by KeyId) =======================

// 0) Caller context
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

// 1) Guard: must have a module selected (category optional)
If(
    IsBlank(varSelectedModuleId),
    Notify("Pick a module first.", NotificationType.Warning),
    With(
        {
            catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value))
        },

        // 2) Snapshot refs for this module (active + all), normalize key as KeyId
        ClearCollect(
            colRefAllByModule,
            AddColumns(
                ShowColumns(
                    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
                    CatItem_ID, Reference_ID, IsActive
                ),
                KeyId, Text(CatItem_ID)
            )
        );
        ClearCollect(
            colRefActiveIds,
            ShowColumns(Filter(colRefAllByModule, IsActive = true), KeyId)
        );

        // 3) Desired selection source:
        //    If user toggled anything (any category), use colToggleLog (Desired=true).
        //    Else, use current category working set where IsSelectedForModule=true.
        ClearCollect(
            colDesiredSel,
            If(
                CountRows(colToggleLog) > 0,
                AddColumns(
                    ShowColumns(Filter(colToggleLog, Desired = true), CatItem_ID),
                    KeyId, Text(CatItem_ID)
                ),
                AddColumns(
                    ShowColumns(Filter(colModuleCatItems_Working, IsSelectedForModule = true), CatItem_ID),
                    KeyId, Text(CatItem_ID)
                )
            )
        );

        // 4) Diffs vs active refs
        ClearCollect(
            colAdds,
            Filter(
                colDesiredSel As d,
                CountIf(colRefActiveIds, KeyId = d.KeyId) = 0
            )
        );
        ClearCollect(
            colRemoves,
            Filter(
                colRefActiveIds As a,
                CountIf(colDesiredSel, KeyId = a.KeyId) = 0
            )
        );

        // 5) Reactivate existing refs for adds (no dupes)
        ForAll(
            colAdds,
            With(
                {
                    refRow: LookUp(colRefAllByModule, KeyId = ThisRecord.KeyId)
                },
                If(
                    !IsBlank(refRow) && refRow.IsActive <> true,
                    Patch(
                        Skill_Matrix_Reference,
                        LookUp(
                            Skill_Matrix_Reference,
                            Mod_ID = varSelectedModuleId && Text(CatItem_ID) = refRow.KeyId
                        ),
                        { IsActive: true, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                    )
                )
            )
        );

        // 6) Build insert rows for adds that still have no ref
        ClearCollect(
            colAdd_NewRows,
            ForAll(
                colAdds,
                With(
                    { existingRef: LookUp(colRefAllByModule, KeyId = ThisRecord.KeyId) },
                    If(
                        IsBlank(existingRef),
                        {
                            Mod_ID:     varSelectedModuleId,
                            CatItem_ID: IfError(Value(ThisRecord.KeyId), ThisRecord.KeyId),
                            IsActive:   true,
                            Created_By: reqUserEmail,
                            Created_At: Now()
                        }
                    )
                )
            )
        );
        // strip blanks
        ClearCollect(colAdd_NewRows, Filter(colAdd_NewRows, !IsBlank(Mod_ID)));

        // 7) Insert new ref rows
        ForAll(
            colAdd_NewRows,
            Patch(Skill_Matrix_Reference, Defaults(Skill_Matrix_Reference), ThisRecord)
        );

        // 8) Soft delete for removals
        ForAll(
            colRemoves,
            With(
                {
                    refRow: LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && Text(CatItem_ID) = ThisRecord.KeyId
                    )
                },
                If(
                    !IsBlank(refRow),
                    Patch(
                        Skill_Matrix_Reference,
                        refRow,
                        { IsActive: false, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                    )
                )
            )
        );

        // 9) Skill-Type edits for current category gallery
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
            colCI_Changed,
            Patch(
                Skill_Matrix_CategoryItems,
                LookUp(Skill_Matrix_CategoryItems, CatItem_ID = ThisRecord.CatItem_ID),
                { Skill_Type: ThisRecord.NewType, Modified_By: reqUserEmail, Modified_At: Now() }
            )
        );

        // 10) Refresh and resnapshot for Reference_ID resolution
        Refresh(Skill_Matrix_Reference);
        ClearCollect(
            colRefAllByModule,
            AddColumns(
                ShowColumns(
                    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
                    CatItem_ID, Reference_ID, IsActive
                ),
                KeyId, Text(CatItem_ID)
            )
        );

        // 11) Counts + code
        Set(varAddCount,    CountRows(colAdds));
        Set(varRemoveCount, CountRows(colRemoves));
        Set(varUpdCount,    CountRows(colCI_Changed));
        Set(varCodeNow,     "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

        // 12) Build Reference_ID payloads
        Set(
            varAddedRefsText,
            Concat(
                AddColumns(colAdds, RefId, LookUp(colRefAllByModule, KeyId = ThisRecord.KeyId, Reference_ID)),
                RefId,
                ";"
            )
        );
        Set(
            varRemovedRefsText,
            Concat(
                AddColumns(colRemoves, RefId, LookUp(colRefAllByModule, KeyId = ThisRecord.KeyId, Reference_ID)),
                RefId,
                ";"
            )
        );

        // 13) Queue Module delta assignment
        If(
            varAddCount + varRemoveCount > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: varCodeNow,
                    Operation: { Value: If(varAddCount>0 && varRemoveCount>0, "ModCatItemAddRem", If(varAddCount>0, "ModCatItemAdded", "ModCatItemRemoved")) },
                    Status: { Value: "Pending" },
                    Module_ID: varSelectedModuleId,
                    Module_Name: varSelectedModuleName,
                    Added_Reference_IDs: varAddedRefsText,
                    Removed_Reference_IDs: varRemovedRefsText,
                    Added_Count: varAddCount,
                    Removed_Count: varRemoveCount,
                    RequestedAt: Now()
                }
            )
        );

        // 14) Queue CatItem_Updated assignment
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

        // 15) Notify
        If(
            (varAddCount + varRemoveCount) > 0 || varUpdCount > 0,
            Notify(
                "Saved. Added: " & varAddCount & " | Removed: " & varRemoveCount & " | Skill-type updates: " & varUpdCount,
                NotificationType.Success
            ),
            Notify("No changes queued.", NotificationType.Information)
        );

        // 16) Rebuild working set (single pass) and clear toggles
        Refresh(Skill_Matrix_Reference);
        ClearCollect(
            colRefActiveIds,
            ShowColumns(
                AddColumns(
                    ShowColumns(
                        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                        CatItem_ID
                    ),
                    KeyId, Text(CatItem_ID)
                ),
                KeyId
            )
        );
        If(
            !IsBlank(varSelectedModuleId) && !IsBlank(catText),
            ClearCollect(
                colModuleCatItems_Working,
                AddColumns(
                    AddColumns(
                        Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                        KeyId, Text(CatItem_ID)
                    ),
                    IsSelectedForModule,
                    CountIf(colRefActiveIds, KeyId = ThisRecord.KeyId) > 0
                )
            ),
            Clear(colModuleCatItems_Working)
        );
        Clear(colToggleLog);

        // 17) Clean temp
        Clear(colAdds); Clear(colRemoves); Clear(colAdd_NewRows); Clear(colDesiredSel); Clear(colCI_Changed); Clear(colRefAllByModule); Clear(colRefActiveIds)
    )
)

