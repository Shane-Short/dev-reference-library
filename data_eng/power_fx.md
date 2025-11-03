If(
    IsBlank(LookUp(colToggleLog, id = ThisItem.id)),
    Collect(colToggleLog, { id: ThisItem.id, Desired: true }),
    UpdateIf(colToggleLog, id = ThisItem.id, { Desired: true })
)



If(
    IsBlank(LookUp(colToggleLog, id = ThisItem.id)),
    Collect(colToggleLog, { id: ThisItem.id, Desired: false }),
    UpdateIf(colToggleLog, id = ThisItem.id, { Desired: false })
)
