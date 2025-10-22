If(
    IsBlank(cmbDelUser.Selected) ||
    (IsBlank(cmbDelUser.Selected.TitleText) && IsBlank(cmbDelUser.Selected.SubtitleText)),
    Reset(cmbDelUser),
    With(
        {
            selEmail: Lower(Text(cmbDelUser.Selected.SubtitleText)),
            selName:  Text(cmbDelUser.Selected.TitleText)
        },
        If(
            !IsBlank(selEmail) &&
            IsBlank(LookUp(colDeleteUsers, Lower(Employee_Email) = selEmail)),
            Collect(colDeleteUsers, { Employee_Email: selEmail, Employee: selName })
        );
        Reset(cmbDelUser)
    )
)
