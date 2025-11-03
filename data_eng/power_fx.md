// --- Final sync: rebuild working set so checkboxes reflect truth ---
// Avoid refresh storms; only refresh once if you must.
// In most cases Patch() updates are reflected locally, so you can skip this.
Refresh(Skill_Matrix_Reference);

// cache the selected category text (robust for Choice/Text)
With(
    { _catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },

    // Build active ref ids for the selected module (normalize CatItem_ID to text)
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
            id, Text(CatItem_ID)    // IMPORTANT: same normalization used in your diff logic
        )
    );

    // If no module or no category, clear the working set to avoid “everything checked”
    If(
        IsBlank(varSelectedModuleId) || IsBlank(_catText),
        Clear(colModuleCatItems_Working),
        // Rebuild the visible category list with the prechecked flag
        With(
            {
                catList:
                    AddColumns(
                        Filter(
                            Skill_Matrix_CategoryItems,
                            Text(Category) = _catText
                        ),
                        id, Text(CatItem_ID)
                    )
            },
            ClearCollect(
                colModuleCatItems_Working,
                AddColumns(
                    catList,
                    IsSelectedForModule,
                    CountIf(colRefActiveIds As r, r.id = id) > 0
                )
            )
        )
    )
);
