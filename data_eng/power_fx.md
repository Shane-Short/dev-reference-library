With(
    {
        src:
            If(
                IsBlank(cmbDelUser.SearchText) || Len(cmbDelUser.SearchText) < 3,
                // 0-row placeholder with correct column shape
                Filter(
                    Table({ DisplayName:"", Mail:"", UserPrincipalName:"" }),
                    false
                ),
                Office365Users.SearchUserV2({ searchTerm: cmbDelUser.SearchText, top: 50 }).value
            )
    },
    AddColumns(
        src,
        "Title",    Coalesce(DisplayName, ""),
        "Subtitle", Coalesce(Mail, UserPrincipalName, "")
    )
)
