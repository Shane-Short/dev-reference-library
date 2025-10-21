// Defensive add → stage into colDeleteUsers (dedupe by email)
If(
    IsBlank(cmbDelUser.Selected) ||
    (IsBlank(cmbDelUser.Selected.DisplayName) &&
     IsBlank(cmbDelUser.Selected.Mail) &&
     IsBlank(cmbDelUser.Selected.UserPrincipalName)),
    Reset(cmbDelUser),
    With(
        {
            selEmail: Lower(Text(Coalesce(cmbDelUser.Selected.Mail, cmbDelUser.Selected.UserPrincipalName))),
            selName:  Text(Coalesce(cmbDelUser.Selected.DisplayName, ""))
        },
        If(
            !IsBlank(selEmail) &&
            IsBlank(LookUp(colDeleteUsers, Lower(Employee_Email) = selEmail)),
            Collect(colDeleteUsers, { Employee_Email: selEmail, Employee: selName })
        );
        Reset(cmbDelUser)
    )
)
