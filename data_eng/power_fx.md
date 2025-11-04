If(
    IsBlank(LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID)),
    Collect(colToggleLog, { CatItem_ID: ThisItem.CatItem_ID, Desired: true }),
    Patch(colToggleLog, LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID), { Desired: true })
)



If(
    IsBlank(LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID)),
    Collect(colToggleLog, { CatItem_ID: ThisItem.CatItem_ID, Desired: false }),
    Patch(colToggleLog, LookUp(colToggleLog, CatItem_ID = ThisItem.CatItem_ID), { Desired: false })
)
