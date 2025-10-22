// Guard against placeholder/blank selection
If(
    IsBlank(cmbDelUser.Selected) ||
    (IsBlank(cmbDelUser.Selected.Title) && IsBlank(cmbDelUser.Selected.Subtitle)),
    Reset(cmbDelUser),
    With(
        {
            // Subtitle is Mail or UPN from our Items AddColumns()
            selEmail: Lower( Text( cmbDelUser.Selected.Subtitle ) ),
            selName:  Text( cmbDelUser.Selected.Title )
        },
        If(
            !IsBlank(selEmail) &&
            IsBlank( LookUp(colDeleteUsers, Lower(Employee_Email) = selEmail) ),
            Collect(
                colDeleteUsers,
                { Employee_Email: selEmail, Employee: selName }
            )
        );
        Reset(cmbDelUser)
    )
)
