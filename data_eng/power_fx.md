With(
    {
        result:
            If(
                !IsBlank(cmbDelUser.SearchText) && Len(cmbDelUser.SearchText) >= 3,
                Office365Users.SearchUserV2({ searchTerm: cmbDelUser.SearchText, top: 50 }).value,
                Table({ DisplayName: "", Mail: "", UserPrincipalName: "" })
            )
    },
    AddColumns(
        result,
        "Title",    Coalesce(DisplayName, ""),
        "Subtitle", Coalesce(Mail, UserPrincipalName, "")
    )
)
