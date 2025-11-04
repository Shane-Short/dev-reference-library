// Guard: must have a module first
If(
    IsBlank(varSelectedModuleId),
    Notify("Pick a module first.", NotificationType.Warning),
    
    With(
        {
            catText: Text(cmbSelectCategory.Selected.Value),

            // 1) Source rows for this category, pre-shaped so columns are guaranteed
            src: ShowColumns(
                    Filter(
                        Skill_Matrix_CategoryItems,
                        Text(Category) = catText
                    ),
                    "CatItem_ID", "Category", "Item", "Skill_Type"
                 )
        },

        // 2) Build the working list:
        //    - id: normalized text key
        //    - IsSelectedForModule: server truth (colRefActiveIds) overridden by any pending toggle (colToggleLog)
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
)



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



