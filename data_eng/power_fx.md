With(
    {
        src:
            If(
                IsBlank(cmbDelUser.SearchText) || Len(cmbDelUser.SearchText) < 3,
                // 0-row placeholder with the same column names the real data has
                Filter(
                    Table({ DisplayName:"", Mail:"", UserPrincipalName:"" }),
                    false
                ),
                Office365Users.SearchUserV2({ searchTerm: cmbDelUser.SearchText, top: 50 }).value
            )
    },
    // Create explicit text columns the Fields panel can bind to reliably
    AddColumns(
        src,
        "Title",      Coalesce(DisplayName, ""),
        "Subtitle",   Coalesce(Mail, UserPrincipalName, "")
    )
)
