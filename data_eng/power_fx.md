If(
    CountRows(colDeleteCatItems) = 0,
    Notify("Select at least one Category — Item to delete.", NotificationType.Information),
    UpdateContext({
        varPendingDeleteAction: "DeleteCatItems",
        varConfirmDeleteMessage:
            "You’re about to queue deletion for " &
            CountRows(colDeleteCatItems) & " CatItem(s). Proceed?",
        varShowConfirmDelete: true
    })
)




// === CASE: DeleteCatItems ===
If(
    varPendingDeleteAction = "DeleteCatItems",
    With(
        {
            req: Office365Users.UserProfileV2(User().Email),
            code: "DelCI-" & Text(Now(), "yyyymmdd-hhnnss")
        },
        ForAll(
            colDeleteCatItems As c,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title:          code,
                    Operation:      { Value: "DeleteCatItem" }, // Choice
                    Status:         { Value: "Pending" },       // Choice
                    CatItem_ID:     c.CatItem_ID,
                    // optional: nice to read in the list
                    Category:       c.Category,
                    Item:           c.Item,
                    RequestedAt:    Now(),
                    RequestedBy: {
                        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                        Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
                        DisplayName:   Coalesce(req.displayName, User().FullName),
                        Email:         Coalesce(req.mail, User().Email),
                        Department:    Coalesce(req.department, ""),
                        JobTitle:      Coalesce(req.jobTitle, ""),
                        Picture:       ""
                    }
                }
            )
        );
        Notify(
            "Queued " & CountRows(colDeleteCatItems) & " CatItem delete request(s).",
            NotificationType.Success
        );
        Clear(colDeleteCatItems);
        Reset(cmbDelCatItem);
        UpdateContext({ varShowConfirmDelete:false, varPendingDeleteAction:"", varConfirmDeleteMessage:"" })
    )
);
