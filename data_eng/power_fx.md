// 1. Recalculate diff right now, before any patches.
// Added = in Working but not in Original
ClearCollect(
    colAddedModules,
    Filter(
        colModulesInPreset_Edit_Working As w,
        IsBlank(
            LookUp(
                colModulesInPreset_Original,
                Module_ID = w.Module_ID
            )
        )
    )
);

// Removed = in Original but not in Working
ClearCollect(
    colRemovedModules,
    Filter(
        colModulesInPreset_Original As o,
        IsBlank(
            LookUp(
                colModulesInPreset_Edit_Working,
                Module_ID = o.Module_ID
            )
        )
    )
);

// DEBUG: temporary, just so we can see if the diff is being found
Notify(
    "DEBUG -> Added:" & CountRows(colAddedModules) &
    " Removed:" & CountRows(colRemovedModules),
    NotificationType.Information
);
