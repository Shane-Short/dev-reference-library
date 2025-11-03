/* --- REBUILD WORKING SET based on current module + category --- */

/* 1) Active Reference rows for this module (ids normalized to text) */
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
        "id", Text(CatItem_ID)
    )
);

/* 2) Category text (handles Choice/Text) */
With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        Clear(colModuleCatItems_Working),
        With(
            {
                /* All CatItems for this category, each with a text id column */
                catList:
                    AddColumns(
                        Filter(
                            Skill_Matrix_CategoryItems,
                            Text(Category) = catText
                        ),
                        "id", Text(CatItem_ID)
                    )
            },
            /* Build working set and compute the selection flag using an *alias* for the refs */
            ClearCollect(
                colModuleCatItems_Working,
                AddColumns(
                    catList,
                    "IsSelectedForModule",
                    CountIf(colRefActiveIds As r, r.id = id) > 0
                )
            )
        )
    )
);
