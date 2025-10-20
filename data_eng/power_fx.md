// Guard against no selection or placeholder rows
If(
    IsBlank(cmbPeople.Selected) ||
    (IsBlank(cmbPeople.Selected.DisplayName) &&
     IsBlank(cmbPeople.Selected.Mail) &&
     IsBlank(cmbPeople.Selected.UserPrincipalName)),
    Reset(cmbPeople),
    With(
        {
            selEmail: Lower(
                Text(Coalesce(cmbPeople.Selected.Mail, cmbPeople.Selected.UserPrincipalName))
            ),
            selName:  Text(Coalesce(cmbPeople.Selected.DisplayName, ""))
        },
        If(
            !IsBlank(selEmail) &&
            IsBlank(
                LookUp(
                    colPresetUsersWorking,
                    Lower(Employee_Email) = selEmail
                )
            ),
            Collect(
                colPresetUsersWorking,
                { Employee_Email: selEmail, Employee: selName }
            )
        );
        Reset(cmbPeople)
    )
)
