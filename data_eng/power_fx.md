// --- REBUILD WORKING SET based on current selections ---

// 2.1 Active Reference rows for this module (normalized to text id)
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

// 2.2 Selected category text (robust against Choice/Text)
With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        Clear(colModuleCatItems_Working),
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



Set(varSelectedModuleId, cmbExistingModule.Selected.Mod_ID);

// Rebuild working set for current module + (possibly already chosen) category
// (Paste the REBUILD WORKING SET block here)



// Rebuild working set for current module + newly chosen category
// (Paste the REBUILD WORKING SET block here)
