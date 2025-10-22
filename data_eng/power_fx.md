If(
    CountRows(colDeleteUsers) = 0,
    Notify("Choose at least one user to delete.", NotificationType.Information),
    UpdateContext({
        varPendingDeleteAction: "DeleteUsers",
        varConfirmDeleteMessage:
            "You’re about to queue deletion for " &
            CountRows(colDeleteUsers) & " user(s)." & Char(10) &
            "This will remove their User Settings rows (and, via Flow, any other cleanup you configure)." & Char(10) &
            Char(10) & "Proceed?",
        varShowConfirmDelete: true
    })
)
