// 6b) Build add/remove diffs from current working set vs active refs
//    (no aliases; use ThisRecord.id so we never reference a missing alias)

ClearCollect(
    colToAdd,
    Filter(
        colModuleCatItems_Working,
        IsSelectedForModule = true &&
        CountIf(colRefActiveIds, id = ThisRecord.id) = 0
    )
);

ClearCollect(
    colToRemove,
    Filter(
        colModuleCatItems_Working,
        IsSelectedForModule = false &&
        CountIf(colRefActiveIds, id = ThisRecord.id) > 0
    )
);






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



// 14) Queue a single “ModuleUpdated” assignment with added/removed Reference_IDs
With(
    {
        codeNow: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss")
    },
    Patch(
        Skill_Matrix_Assignments,
        Defaults(Skill_Matrix_Assignments),
        {
            Title: codeNow,
            Operation: { Value: "ModuleUpdated" },
            Status: { Value: "Pending" },
            Module_ID: varSelectedModuleId,
            Module_Name: varSelectedModuleName,

            Added_Reference_IDs: Concat(
                ForAll(
                    Coalesce(colToAdd, Table()),
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && Text(CatItem_ID) = Text(ThisRecord.CatItem_ID),
                        Reference_ID
                    )
                ),
                Text(Result),
                ";"
            ),

            Removed_Reference_IDs: Concat(
                ForAll(
                    Coalesce(colToRemove, Table()),
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && Text(CatItem_ID) = Text(ThisRecord.CatItem_ID),
                        Reference_ID
                    )
                ),
                Text(Result),
                ";"
            ),

            Added_Count:   varAddCount,
            Removed_Count: varRemoveCount,
            Updated_Count: varUpdateCount,
            RequestedAt: Now(),
            RequestedBy: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                Claims: "i:0#.f|membership|" & User().Email,
                DisplayName: User().FullName,
                Email: User().Email
            }
        }
    )
);



Set(varAddCount,    CountRows(Coalesce(colToAdd, Table())));
Set(varRemoveCount, CountRows(Coalesce(colToRemove, Table())));
Set(varUpdateCount, CountRows(Coalesce(colTypeChanges, Table()))); // set to 0 if you don’t track type changes






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
