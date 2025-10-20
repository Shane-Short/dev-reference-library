With(
    {
        p: If(IsEmpty(cmbPeople.SelectedItems), Blank(), First(cmbPeople.SelectedItems)),
        _email: Lower(Coalesce(p.Mail, p.UserPrincipalName)),
        _name: Coalesce(p.DisplayName, "")
    },
    If(
        !IsBlank(p) &&
        !IsBlank(_email) &&
        IsBlank(LookUp(colPresetUsersWorking, Lower(Employee_Email) = _email)),
        Collect(colPresetUsersWorking, { Employee_Email: _email, Employee: _name })
    )
);
Reset(cmbPeople)
