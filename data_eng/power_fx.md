// fresh toggle cache
Clear(colToggleLog);

// capture module context
Set(varSelectedModuleId, cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// active refs for this module (only CatItem_ID; keep it simple)
ClearCollect(
    colRefActiveIds,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
        "CatItem_ID"
    )
);

// if a category is already chosen, rebuild the working list; else clear it
If(
    IsBlank(cmbSelectCategory.Selected),
    Clear(colModuleCatItems_Working),
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                // normalize CatItem_ID as text so comparisons are stable
                AddColumns(
                    Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                    id, Text(CatItem_ID)
                ),
                IsSelectedForModule,
                // direct lookup into colRefActiveIds by CatItem_ID
                !IsBlank(
                    LookUp(colRefActiveIds, Text(CatItem_ID) = id)
                )
            )
        )
    )
);







// fresh toggle cache
Clear(colToggleLog);

// need a module to decide selection-state; else just show nothing
If(
    IsBlank(varSelectedModuleId) || IsBlank(cmbSelectCategory.Selected),
    Clear(colModuleCatItems_Working),
    With(
        {
            catText: Text(cmbSelectCategory.Selected.Value)
        },
        // recompute active refs (cheap; ensures order of ops is correct)
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
                !IsBlank(
                    LookUp(colRefActiveIds, Text(CatItem_ID) = id)
                )
            )
        )
    )
);










