// ======================= Save Module Config (robust) =======================

// 0) Caller context
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

// 1) Guard: must have a module selected (category optional now)
If(
    IsBlank(varSelectedModuleId),
    Notify("Pick a module first.", NotificationType.Warning),
    With(
        {
            // current category text (may be Blank())
            catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value))
        },

        // 2) Snapshot refs for this module (active + all)
        ClearCollect(
            colRefAllByModule,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
                "CatItem_ID", "Reference_ID", "IsActive"
            )
        );
        ClearCollect(
            colRefActiveIds,
            AddColumns(
                ShowColumns(
                    Filter(colRefAllByModule, IsActive = true),
                    "CatItem_ID"
                ),
                id, Text(CatItem_ID)
            )
        );

        // 3) Build desired selection (global): if user toggled anything anywhere, use that; else use current category working list
        ClearCollect(
            colDesiredSel,
            If(
                CountRows(colToggleLog) > 0,
                AddColumns(
                    ShowColumns(Filter(colToggleLog, Desired = true), "CatItem_ID"),
                    id, Text(CatItem_ID)
                ),
                ShowColumns(Filter(colModuleCatItems_Working, IsSelectedForModule = true), "id")
            )
        );

        // 4) Compute diffs vs active refs (global)
        ClearCollect(
            colAdds,
            Filter(
                colDesiredSel As d,
                CountIf(colRefActiveIds, id = d.id) = 0
            )
        );
        ClearCollect(
            colRemoves,
            Filter(
                colRefActiveIds As a,
                CountIf(colDesiredSel, id = a.id) = 0
            )
        );

        // 5) Reactivate any existing refs for adds (no dupes)
        ForAll(
            colAdds,
            With(
                {
                    refRow: LookUp(colRefAllByModule, Text(CatItem_ID) = ThisRecord.id)
                },
                If(
                    !IsBlank(refRow) && refRow.IsActive <> true,
                    Patch(
                        Skill_Matrix_Reference,
                        LookUp(
                            Skill_Matrix_Reference,
                            Mod_ID = varSelectedModuleId && CatItem_ID = refRow.CatItem_ID
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
                    {
                        existingRef: LookUp(colRefAllByModule, Text(CatItem_ID) = ThisRecord.id)
                    },
                    If(
                        IsBlank(existingRef),
                        {
                            Mod_ID:     varSelectedModuleId,
                            CatItem_ID: IfError(Value(ThisRecord.id), ThisRecord.id),
                            IsActive:   true,
                            CreatedBy:  reqUserEmail,
                            CreatedAt:  Now()
                        }
                    )
                )
            )
        );
        // remove blanks
        ClearCollect(colAdd_NewRows, Filter(colAdd_NewRows, !IsBlank(Mod_ID)));

        // 7) Insert new ref rows
        ForAll(
            colAdd_NewRows,
            Patch(Skill_Matrix_Reference, Defaults(Skill_Matrix_Reference), ThisRecord)
        );

        // 8) Apply removals (soft delete)
        ForAll(
            colRemoves,
            With(
                {
                    refRow: LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && Text(CatItem_ID) = ThisRecord.id
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

        // 9) Skill-Type edits (current category gallery)
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
                { Skill_Type: ThisRecord.NewType, ModifiedBy: reqUserEmail, ModifiedAt: Now() }
            )
        );

        // 10) Re-snapshot refs to resolve Reference_IDs for assignment payloads
        Refresh(Skill_Matrix_Reference);
        ClearCollect(
            colRefAllByModule,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
                "CatItem_ID", "Reference_ID", "IsActive"
            )
        );

        // 11) Counts
        Set(varAddCount,    CountRows(colAdds));
        Set(varRemoveCount, CountRows(colRemoves));
        Set(varUpdCount,    CountRows(colCI_Changed));
        Set(varCodeNow,     "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

        // 12) Build Added_/Removed_Reference_IDs strings
        Set(
            varAddedRefsText,
            Concat(
                AddColumns(
                    colAdds,
                    RefId, LookUp(colRefAllByModule, Text(CatItem_ID) = ThisRecord.id, Reference_ID)
                ),
                RefId,
                ";"
            )
        );
        Set(
            varRemovedRefsText,
            Concat(
                AddColumns(
                    colRemoves,
                    RefId, LookUp(colRefAllByModule, Text(CatItem_ID) = ThisRecord.id, Reference_ID)
                ),
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

        // 16) Rebuild working set (single pass) + clear pending toggles
        Refresh(Skill_Matrix_Reference);
        ClearCollect(
            colRefActiveIds,
            AddColumns(
                ShowColumns(
                    Filter(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && IsActive = true
                    ),
                    "CatItem_ID"
                ),
                id, Text(CatItem_ID)
            )
        );
        If(
            !IsBlank(varSelectedModuleId) && !IsBlank(catText),
            ClearCollect(
                colModuleCatItems_Working,
                AddColumns(
                    AddColumns(
                        Filter(
                            Skill_Matrix_CategoryItems,
                            Text(Category) = catText
                        ),
                        id, Text(CatItem_ID)
                    ),
                    IsSelectedForModule,
                    CountIf(colRefActiveIds, id = ThisRecord.id) > 0
                )
            ),
            Clear(colModuleCatItems_Working)
        );
        Clear(colToggleLog);

        // 17) Clean temp
        Clear(colAdds); Clear(colRemoves); Clear(colAdd_NewRows); Clear(colDesiredSel); Clear(colCI_Changed); Clear(colRefAllByModule)
    )
)














// Pending selection overrides server truth; otherwise use server truth
If(
    !IsBlank(LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID)),
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID).Desired,
    IsSelectedForModule
)




Patch(
    colToggleLog,
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID),
    { CatItem_ID: ThisItem.CatItem_ID, Desired: true }
);









Patch(
    colToggleLog,
    LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID),
    { CatItem_ID: ThisItem.CatItem_ID, Desired: false }
);







Set(varSelectedModuleId,   cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// clear any pending toggles when switching modules
Clear(colToggleLog);

// rebuild working set if a category is selected
If(
    !IsBlank(cmbSelectCategory.Selected),
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },
        ClearCollect(
            colRefActiveIds,
            AddColumns(
                ShowColumns(
                    Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                    "CatItem_ID"
                ),
                id, Text(CatItem_ID)
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
                CountIf(colRefActiveIds, id = ThisRecord.id) > 0
            )
        )
    ),
    Clear(colModuleCatItems_Working)
);






// rebuild working set for this category
With(
    { catText: Text(cmbSelectCategory.Selected.Value) },
    ClearCollect(
        colRefActiveIds,
        AddColumns(
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
                "CatItem_ID"
            ),
            id, Text(CatItem_ID)
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
            CountIf(colRefActiveIds, id = ThisRecord.id) > 0
        )
    )
);








Clear(colToggleLog);
Clear(colModuleCatItems_Working);
Clear(colRefActiveIds);
Set(varSelectedModuleId,   Blank());
Set(varSelectedModuleName, Blank());
