// Guard: must have a module picked first
If(
    IsBlank(varSelectedModuleId),
    Notify("Pick a module first.", NotificationType.Warning),
    /* ELSE */
    ClearCollect(
        colModuleCatItems_Working,
        With(
            {
                selectedCat: Text(cmbSelectCategory.Selected.Value),

                // 1) Clean source rows for this category (fixed schema)
                src: ShowColumns(
                        Filter(
                            Skill_Matrix_CategoryItems,
                            Text(Category) = selectedCat
                        ),
                        "CatItem_ID", "Category", "Item", "Skill_Type"
                    )
            },
            // 2) Build working list:
            //    - id: normalized text key
            //    - IsSelectedForModule: server truth overridden by any pending toggle
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
