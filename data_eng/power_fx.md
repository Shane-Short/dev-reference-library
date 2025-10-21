// ------------------------------
// Assign/Unassign button - OnSelect
// ------------------------------

// Reset modal state first (safety)
UpdateContext({
    varShowConfirm:    false,
    varPendingAction:  "",
    varConfirmMessage: "",
    varAssignSummary:  ""
});

// ===== Guards =====
If(
    IsBlank(varPresetName) || IsBlank(varPresetId),
    Notify("Choose a team preset.", NotificationType.Warning),
    If(
        IsEmpty(colPresetUsersWorking),
        Notify("Add at least one user.", NotificationType.Warning),

        // ===== Build diffs: Added vs Removed =====
        With(
            {
                _added:
                    Filter(
                        colPresetUsersWorking As w,
                        CountIf(
                            colPresetUsersOriginal,
                            Lower(Employee_Email) = Lower(w.Employee_Email)
                        ) = 0
                    ),
                _removed:
                    Filter(
                        colPresetUsersOriginal As o,
                        CountIf(
                            colPresetUsersWorking,
                            Lower(Employee_Email) = Lower(o.Employee_Email)
                        ) = 0
                    )
            },

            // materialize diffs into collections we can show/patch later
            ClearCollect(colAddedUsers,   _added);
            ClearCollect(colRemovedUsers, _removed);

            // If nothing changed, short-circuit with info
            If(
                CountRows(colAddedUsers) = 0 && CountRows(colRemovedUsers) = 0,
                Notify("No changes to queue for this preset.", NotificationType.Information),

                // ===== Build summary text (first few examples) =====
                With(
                    {
                        maxShow: 5,
                        addCount: CountRows(colAddedUsers),
                        remCount: CountRows(colRemovedUsers)
                    },
                    UpdateContext({
                        varAssignSummary:
                            "Preset: " & varPresetName & Char(10) &
                            "Add user(s): " & addCount & Char(10) &
                            Concat(
                                FirstN(colAddedUsers, Min(maxShow, addCount)),
                                "- " & Coalesce(Employee, "") & " (" & Employee_Email & ")" & Char(10)
                            ) &
                            If(addCount > maxShow, "... and " & Text(addCount - maxShow) & " more" & Char(10), "") &
                            Char(10) &
                            "Remove user(s): " & remCount & Char(10) &
                            Concat(
                                FirstN(colRemovedUsers, Min(maxShow, remCount)),
                                "- " & Coalesce(Employee, "") & " (" & Employee_Email & ")" & Char(10)
                            ) &
                            If(remCount > maxShow, "... and " & Text(remCount - maxShow) & " more", "")
                    })
                );

                // ===== Open confirm modal =====
                UpdateContext({
                    varPendingAction: "AssignUnassign",
                    varConfirmMessage: varAssignSummary,
                    varShowConfirm: true
                })
            )
        ) // end With diffs
    )
);
