With(
    {
        selEmail: Lower(cmbPeople.Selected.Mail),
        selName:  Coalesce(
            cmbPeople.Selected.DisplayName,
            cmbPeople.Selected.GivenName & " " & cmbPeople.Selected.Surname
        )
    },
    If(
        !IsBlank(selEmail) &&
        IsBlank(LookUp(colPresetUsersWorking, Lower(Employee_Email) = selEmail)),
        Collect(colPresetUsersWorking, { Employee_Email: selEmail, Employee: selName })
    )
);
Reset(cmbPeople)
