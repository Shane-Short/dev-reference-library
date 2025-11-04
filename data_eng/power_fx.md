// Rebuild active CatItem IDs for this module (no Distinct/Result/Value shenanigans)
ClearCollect(
    colRefActiveIds,
    RenameColumns(
        ShowColumns(
            Filter(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && IsActive = true
            ),
            "CatItem_ID"
        ),
        "CatItem_ID", "id"   // => colRefActiveIds has a single column: id (Text)
    )
);



With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        FirstN(Skill_Matrix_CategoryItems, 0),
        AddColumns(
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Text(Category) = catText
                ),
                id, Text(CatItem_ID)
            ),
            IsSelectedForModule,
            If(
                IsBlank(id),
                false,
                CountIf(colRefActiveIds, id = ThisRecord.id) > 0
            )
        )
    )
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




// set module context
Set(varSelectedModuleId,   cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// clear in-progress toggles whenever module changes
Clear(colToggleLog);

// rebuild active IDs for this module (A)
ClearCollect(
    colRefActiveIds,
    With(
        {
            src:
                ShowColumns(
                    Filter(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && IsActive = true
                    ),
                    CatItem_ID
                ),
            d: Distinct(
                src,
                Text(CatItem_ID)
            )
        },
        AddColumns(d, id, Text(Coalesce(Result, Value)))
    )
);

// if a category is already chosen, the gallery formula (B) will use colRefActiveIds;
// nothing else needed here





// cache the selected category text
Set(varSelectedCategoryText, If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)));

// clear in-progress toggles whenever category changes
Clear(colToggleLog);

// refresh active IDs for this module (A)
ClearCollect(
    colRefActiveIds,
    With(
        {
            src:
                ShowColumns(
                    Filter(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && IsActive = true
                    ),
                    CatItem_ID
                ),
            d: Distinct(
                src,
                Text(CatItem_ID)
            )
        },
        AddColumns(d, id, Text(Coalesce(Result, Value)))
    )
);

// gallery Items (B) will now reflect correctly using colRefActiveIds + this category





Refresh(Skill_Matrix_Reference);

// rebuild colRefActiveIds (A)
ClearCollect(
    colRefActiveIds,
    With(
        {
            src:
                ShowColumns(
                    Filter(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && IsActive = true
                    ),
                    CatItem_ID
                ),
            d: Distinct(
                src,
                Text(CatItem_ID)
            )
        },
        AddColumns(d, id, Text(Coalesce(Result, Value)))
    )
);

// clear the toggle log now that server state is authoritative
Clear(colToggleLog);

// (no need to touch the gallery; its Items (B) will re-evaluate automatically)
