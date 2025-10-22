AddColumns(
    Office365Users.SearchUserV2({ searchTerm: "a", top: 10 }).value,
    "Title",    Coalesce(DisplayName, ""),
    "Subtitle", Coalesce(Mail, UserPrincipalName, "")
)
