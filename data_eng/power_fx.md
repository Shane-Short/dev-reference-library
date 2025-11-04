RequestedBy: {
    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
    Claims: "i:0#.f|membership|" & Lower(reqUserEmail),
    DisplayName: reqUserName,
    Email: Lower(reqUserEmail)
}
