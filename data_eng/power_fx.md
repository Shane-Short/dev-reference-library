With(
    {
        src:
            If(
                IsBlank(Self.SearchText) || Len(Self.SearchText) < 3,
                // keep a consistent schema when empty
                Table({ DisplayName:"", Mail:"", UserPrincipalName:"" }),
                Office365Users.SearchUserV2({ searchTerm: Self.SearchText, top: 50 }).value
            )
    },
    AddColumns(
        src,
        "Title",    Coalesce(DisplayName, ""),
        "Subtitle", Coalesce(Mail, UserPrincipalName, "")
    )
)

// Guard against no selection / placeholder row
If(
    IsBlank(cmbUsers.Selected) ||
    (IsBlank(cmbUsers.Selected.Title) && IsBlank(cmbUsers.Selected.Subtitle)),
    Reset(cmbUsers),
    With(
        {
            selEmail: Lower( Text( cmbUsers.Selected.Subtitle ) ),
            selName:  Text( cmbUsers.Selected.Title )
        },
        If(
            !IsBlank(selEmail) &&
            IsBlank( LookUp(colPresetUsersWorking, Lower(Employee_Email) = selEmail) ),
            Collect(colPresetUsersWorking, { Employee_Email: selEmail, Employee: selName })
        );
        Reset(cmbUsers)
    )
)
