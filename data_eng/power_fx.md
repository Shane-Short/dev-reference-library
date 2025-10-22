If(
    IsBlank(cmbDelUser.SearchText) || Len(cmbDelUser.SearchText) < 3,
    // 0-row placeholder with the right columns
    Filter(Table({ DisplayName:"", Mail:"", UserPrincipalName:"" }), false),
    Office365Users.SearchUserV2({ searchTerm: cmbDelUser.SearchText, top: 50 }).value
)
