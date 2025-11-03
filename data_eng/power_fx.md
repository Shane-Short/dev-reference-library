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

// 6b) Make a small collection of NEW refs to insert (no direct ForAll over list)
Clear(colAdd_NewRows);
ForAll(
    Filter(
        colAdds As a,
        IsBlank(
            LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id)
        )
    ),
    With(
        {
            ci: LookUp(Skill_Matrix_CategoryItems, Text(CatItem_ID) = a.id),
            newRefId: GUID()
        },
        Collect(
            colAdd_NewRows,
            {
                Reference_ID: newRefId,
                Mod_ID: varSelectedModuleId,
                Module: varSelectedModuleName,
                CatItem_ID: ci.CatItem_ID,
                Category: ci.Category,
                Item: ci.Item,
                SkillLevel: ci.Skill_Type,   // (your Reference column name)
                IsActive: true,
                Created_By: reqUserEmail,
                Created_At: Now()
            }
        )
    )
);

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

)) // end With (catText) and outer If






// After you set varSelectedModuleId & varSelectedModuleName
ClearCollect(
    colPendingSel,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            CatItem_ID
        ),
        IsSelected, true
    )
);

// Build colRefActiveIds for this module (used by Defaults below)
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

// Optional: clear the working list until a category is picked
Clear(colModuleCatItems_Working);
Reset(cmbSelectCategory);







// Rebuild visible list – merge base state with pending overrides
With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        Clear(colModuleCatItems_Working),
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
                    With(
                        {
                            baseSel: CountIf(colRefActiveIds As r, r.id = id) > 0,
                            pend:    LookUp(colPendingSel, CatItem_ID = CatItem_ID, IsSelected)
                        },
                        Coalesce(pend, baseSel)
                    )
                )
            )
        )
    )
);







With(
  {
    baseSel: CountIf(colRefActiveIds, Text(CatItem_ID) = Text(ThisItem.CatItem_ID)) > 0,
    pend:    LookUp(colPendingSel, CatItem_ID = ThisItem.CatItem_ID, IsSelected)
  },
  Coalesce(pend, baseSel)
)






If(
  IsBlank(LookUp(colPendingSel, CatItem_ID = ThisItem.CatItem_ID)),
  Collect(colPendingSel, { CatItem_ID: ThisItem.CatItem_ID, IsSelected: true }),
  UpdateIf(colPendingSel, CatItem_ID = ThisItem.CatItem_ID, { IsSelected: true })
);






If(
  IsBlank(LookUp(colPendingSel, CatItem_ID = ThisItem.CatItem_ID)),
  Collect(colPendingSel, { CatItem_ID: ThisItem.CatItem_ID, IsSelected: false }),
  UpdateIf(colPendingSel, CatItem_ID = ThisItem.CatItem_ID, { IsSelected: false })
);





// seed from current truth so the UI defaults match the list
ClearCollect(colRefActiveIds,
    AddColumns(
        ShowColumns(Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true), CatItem_ID),
        id, Text(CatItem_ID)
    )
);
ClearCollect(colPendingSel,
    AddColumns(
        ShowColumns(Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true), CatItem_ID),
        IsSelected, true
    )
);
