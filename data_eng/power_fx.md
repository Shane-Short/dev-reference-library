If(
    !IsBlank(cmbDelUser.SearchText) && Len(cmbDelUser.SearchText) >= 3,
    AddColumns(
        Office365Users.SearchUserV2({ searchTerm: cmbDelUser.SearchText, top: 50 }).value,
        "Title",    Coalesce(DisplayName, ""),
        "Subtitle", Coalesce(Mail, UserPrincipalName, "")
    ),
    []
)
