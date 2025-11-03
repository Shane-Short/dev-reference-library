// 6a.1) Build rows we must insert (for ids in colAdds that lacked an existing ref)
ClearCollect(
    colAdd_NewRows,
    ForAll(
        colAdds As a,
        With(
            { existingRef: LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id) },
            If(
                IsBlank(existingRef),
                {
                    Mod_ID:     varSelectedModuleId,
                    CatItem_ID: IfError(Value(a.id), a.id),   // works whether CatItem_ID is Number or Text
                    IsActive:   true,
                    CreatedBy:  reqUserEmail,
                    CreatedAt:  Now()
                }
            )
        )
    )
);
// Remove blanks (when existingRef was found)
ClearCollect(colAdd_NewRows, Filter(colAdd_NewRows, !IsBlank(Mod_ID)));




// 12.1 Refresh Reference so UI reflects truth post-save
Refresh(Skill_Matrix_Reference);

// 12.2 Recompute active Reference IDs for this module
ClearCollect(
    colRefActiveIds,
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
);

// 12.3 Rebuild the category’s working list with correct check-state
If(
    IsBlank(varSelectedModuleId) || IsBlank(cmbSelectCategory.Selected),
    Clear(colModuleCatItems_Working),
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },
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
        )
    )
);








Set(varAddCount,    CountRows(Coalesce(colToAdd, Table())));
Set(varRemoveCount, CountRows(Coalesce(colToRemove, Table())));
Set(varUpdateCount, CountRows(Coalesce(colTypeChanges, Table())));









Set(varUpdCount,     CountRows(colCI_Changed));









// cache refs for this module once
ClearCollect(
    colRefAllByModule,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
        "CatItem_ID", "Reference_ID", "IsActive"
    )
);








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








// 4) Desired selection from global toggle log (all categories)
ClearCollect(
    colDesiredSel,
    AddColumns(
        ShowColumns(Filter(colToggleLog, Desired = true), "CatItem_ID"),
        id, Text(CatItem_ID)
    )
);

// 5) Global diffs vs current active refs (no cat filter)
ClearCollect(
    colAdds,
    Filter(colDesiredSel As d, CountIf(colRefActiveIds, id = d.id) = 0)
);

ClearCollect(
    colRemoves,
    Filter(colRefActiveIds As a, CountIf(colDesiredSel, id = a.id) = 0)
);











Clear(colToggleLog);







// ======================= Save Module Config (robust) =======================

// 0) Basic user cache
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

// 1) Guard: must have a module and a category
If(
    IsBlank(varSelectedModuleId) || IsBlank(cmbSelectCategory.Selected),
    Notify("Pick a module and a category first.", NotificationType.Warning),
    /* ELSE: do the save work */ With(
        { catText: Text(cmbSelectCategory.Selected.Value) },

// 2) Snapshot: active refs for this module (id normalized to text)
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            CatItem_ID
        ),
        id, Text(CatItem_ID)
    )
);

// 3) The category’s CatItems (normalized id)
With(
{ catList:
    AddColumns(
        Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
        id, Text(CatItem_ID)
    )
},

// 4) Desired selection: what the user marked in the working list
ClearCollect(
    colDesiredSel,
    ShowColumns(
        Filter(colModuleCatItems_Working, IsSelectedForModule = true),
        "id"
    )
);

// 5) Compute diffs (adds/removes) using ONLY local collections
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
        CountIf(catList, id = a.id) > 0      // only this category
        && CountIf(colDesiredSel, id = a.id) = 0
    )
);

// 6) APPLY ADDS (reactivate if exists, else create)
// 6a) Reactivate existing refs for any add that already has a ref row
ClearCollect(
    colAdd_Reactivations,
    ForAll(
        colAdds As a,
        With(
            {
                refRow: LookUp(
                    Skill_Matrix_Reference,
                    Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id
                )
            },
            If(
                !IsBlank(refRow),
                Patch(
                    Skill_Matrix_Reference,
                    refRow,
                    { IsActive: true, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                )
            )
        )
    )
);

// 6a.1) Build rows we must insert (for ids in colAdds that lacked an existing ref)
ClearCollect(
    colAdd_NewRows,
    ForAll(
        colAdds As a,
        With(
            { existingRef: LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id) },
            If(
                IsBlank(existingRef),
                {
                    Mod_ID:     varSelectedModuleId,
                    CatItem_ID: IfError(Value(a.id), a.id),   // works whether CatItem_ID is Number or Text
                    IsActive:   true,
                    CreatedBy:  reqUserEmail,
                    CreatedAt:  Now()
                }
            )
        )
    )
);
// Remove blanks (when existingRef was found)
ClearCollect(colAdd_NewRows, Filter(colAdd_NewRows, !IsBlank(Mod_ID)));


// 6c) Insert the new rows safely (iterate the collection, Patch Defaults)
ForAll(
    colAdd_NewRows As r,
    Patch(
        Skill_Matrix_Reference,
        Defaults(Skill_Matrix_Reference),
        r
    )
);

// 7) APPLY REMOVES (soft-delete)
ForAll(
    colRemoves As r,
    With(
        {
            refRow: LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && Text(CatItem_ID) = r.id
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

// 8) Skill_Type edits (persist changes in CategoryItems only)
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
        { Skill_Type: c.NewType, ModifiedBy: reqUserEmail, ModifiedAt: Now() }
    )
);

// 9) Counts + assignment logs (NO person blob unless you truly have that column)
Set(varAddCount,     CountRows(colAdds));
Set(varRemoveCount,  CountRows(colRemoves));
Set(varUpdCount,     CountRows(colCI_Changed));
Set(varCodeNow,      "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

// 9.1 Module delta log
If(
    varAddCount + varRemoveCount > 0,
    With(
        {
            addedRefsText:   Concat(
                                ShowColumns(
                                  AddColumns(colAdds, Reference_ID,
                                    Coalesce(
                                      LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = id).Reference_ID,
                                      LookUp(colAdd_NewRows, Text(CatItem_ID) = id).Reference_ID
                                    )
                                  ),
                                  "Reference_ID"
                                ),
                                Reference_ID,
                                ";"
                              ),
            removedRefsText: Concat(
                                ShowColumns(
                                  AddColumns(colRemoves, Reference_ID,
                                    LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = id).Reference_ID
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
                // If you DO have a Person column called RequestedBy, use the object below.
                // RequestedBy: {
                //   '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                //   Claims: "i:0#.f|membership|" & Lower(reqUserEmail),
                //   DisplayName: reqUserName,
                //   Email: reqUserEmail
                // }
            }
        )
    )
);

// 9.2 Skill type change log
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
            // RequestedBy: { ... }  // same optional person object as above
        }
    )
);

// 10) Notify
If(
    (varAddCount + varRemoveCount) > 0 || varUpdCount > 0,
    Notify(
        "Saved. Added: " & varAddCount & " | Removed: " & varRemoveCount & " | Skill-type updates: " & varUpdCount,
        NotificationType.Success
    ),
    Notify("No changes queued.", NotificationType.Information)
);

// 11) Rebuild working set so checkboxes reflect truth immediately
//    (one refresh is enough; the rest is local)
Refresh(Skill_Matrix_Reference);
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            CatItem_ID
        ),
        id, Text(CatItem_ID)
    )
);
With(
    {
        catList:
            AddColumns(
                Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                id, Text(CatItem_ID)
            )
    },
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            catList,
            IsSelectedForModule,
            CountIf(colRefActiveIds, id = ThisRecord.id) > 0
        )
    )
);

// 12) Clean temp
Clear(colAdds); Clear(colRemoves); Clear(colAdd_NewRows); Clear(colDesiredSel); Clear(colCI_Changed)

// 12.1 Refresh Reference so UI reflects truth post-save
Refresh(Skill_Matrix_Reference);

// 12.2 Recompute active Reference IDs for this module
ClearCollect(
    colRefActiveIds,
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
);

// 12.3 Rebuild the category’s working list with correct check-state
If(
    IsBlank(varSelectedModuleId) || IsBlank(cmbSelectCategory.Selected),
    Clear(colModuleCatItems_Working),
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },
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
        )
    )
);

Set(varAddCount,    CountRows(Coalesce(colToAdd, Table())));
Set(varRemoveCount, CountRows(Coalesce(colToRemove, Table())));
Set(varUpdateCount, CountRows(Coalesce(colTypeChanges, Table()))); // set to 0 if you don’t track type changes


