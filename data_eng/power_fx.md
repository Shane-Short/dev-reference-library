With(
    { sel: First(cmbPresetModules.SelectedItems) },
    Concatenate(
        "FIELDS: ",
        Concat(
            Table(
                "Title=" & Coalesce(sel.Title, "␀"),
                "Modules=" & Coalesce(sel.Modules, "␀"),
                "Module_ID=" & Coalesce(sel.Module_ID, "␀"),
                "Mod_ID=" & Coalesce(sel.Mod_ID, "␀"),
                "Name=" & Coalesce(sel.Name, "␀")
            ),
            Value & " | "
        )
    )
)
