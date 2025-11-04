// Category picker OnChange

If(
    IsBlank(varSelectedModuleId) || IsBlank(cmbSelectCategory.Selected),
    Notify("Pick a module first, then a category.", NotificationType.Warning),

    /* else */
    Set(varSelectedCategoryText, Text(cmbSelectCategory.Selected.Value));

    // (Assumes colRefActiveIds was built in Module picker OnChange.
    // If you want this control to be self-sufficient, uncomment the next block.)
    /*
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
    */

    // Build the working list for the selected category
    With(
        {
            src:
                ShowColumns(
                    Filter(
                        Skill_Matrix_CategoryItems,
                        Text(Category) = varSelectedCategoryText
                    ),
                    CatItem_ID, Category, Item, Skill_Type
                )
        },
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                // give each row a stable text id
                AddColumns(src, id, Text(CatItem_ID)),
                // final checkbox state = toggle override (if any) else active-ref membership
                IsSelectedForModule,
                    Coalesce(
                        // IMPORTANT: qualify CatItem_ID with ThisRecord
                        LookUp(
                            colToggleLog,
                            CatItem_ID = ThisRecord.CatItem_ID
                        ).Desired,
                        // membership in active refs for this module
                        !IsBlank(
                            LookUp(
                                colRefActiveIds,
                                id = ThisRecord.id
                            )
                        )
                    )
            )
        )
    )
);
