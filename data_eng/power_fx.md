// 1) Skip if no module chosen yet
If(IsBlank(varSelectedModuleId), Return());

// 2) Resolve category text safely
With(
    { catText: Text(cmbSelectCategory.Selected.Value) },

    // 3) Build the working list for THIS category
    //    - id: normalized CatItem_ID as text
    //    - IsSelectedForModule: overlay server state (colRefActiveIds) with any pending toggles (colToggleLog)
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            Filter(
                Skill_Matrix_CategoryItems,
                Text(Category) = catText
            ),
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
);


// Upsert: Desired = true for this CatItem_ID
If(
    IsBlank(LookUp(colToggleLog, CatItem_ID = ThisItem.id)),
    Collect(colToggleLog, { CatItem_ID: ThisItem.id, Desired: true }),
    Patch(colToggleLog, LookUp(colToggleLog, CatItem_ID = ThisItem.id), { Desired: true })
);


// Upsert: Desired = false for this CatItem_ID
If(
    IsBlank(LookUp(colToggleLog, CatItem_ID = ThisItem.id)),
    Collect(colToggleLog, { CatItem_ID: ThisItem.id, Desired: false }),
    Patch(colToggleLog, LookUp(colToggleLog, CatItem_ID = ThisItem.id), { Desired: false })
);
