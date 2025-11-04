// Guard: must have a module picked first
If(
    IsBlank(varSelectedModuleId),
    Notify("Pick a module first.", NotificationType.Warning),
    
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },

        // Build the working list for THIS category
        // - id: normalized CatItem_ID as text
        // - IsSelectedForModule: server truth (colRefActiveIds) overlaid by any pending toggles (colToggleLog)
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems As ci,
                    Text(ci.Category) = catText
                ),
                id, Text(ci.CatItem_ID),
                IsSelectedForModule,
                    With(
                        { tgl: LookUp(colToggleLog, CatItem_ID = ci.CatItem_ID) },
                        If(
                            IsBlank(tgl),
                            CountIf(
                                colRefActiveIds,
                                Text(CatItem_ID) = Text(ci.CatItem_ID)
                            ) > 0,
                            tgl.Desired
                        )
                    )
            )
        )
    )
)
