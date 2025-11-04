// Cache robust category text once (handles Choice/Result/SelectedText)
Set(
    varSelectedCategoryText,
    Text(
        Coalesce(
            cmbSelectCategory.Selected.Value,
            cmbSelectCategory.Selected.Result,
            cmbSelectCategory.SelectedText.Value
        )
    )
);

// Require a module + category
If(
    IsBlank(varSelectedModuleId) || IsBlank(varSelectedCategoryText),
    Notify("Pick a module and a category first.", NotificationType.Warning),
    With(
        {
            // minimal, clean source set for the category
            src: ShowColumns(
                    Filter(
                        Skill_Matrix_CategoryItems,
                        Text(Category) = varSelectedCategoryText
                    ),
                    "CatItem_ID", "Category", "Item", "Skill_Type"
                )
        },
        // Build the working rows: normalized id + selected flag
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                src,
                id, Text(CatItem_ID),
                IsSelectedForModule,
                    With(
                        { tgl: LookUp(colToggleLog, CatItem_ID = CatItem_ID) },
                        If(
                            IsBlank(tgl),
                            CountIf(colRefActiveIds, id = Text(CatItem_ID)) > 0,
                            tgl.Desired
                        )
                    )
            )
        )
    )
);
