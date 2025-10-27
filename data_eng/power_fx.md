// 2.0) Ensure we have a Module row and Module_ID (varModuleIdSafe)
With(
    {
        _typedName: Trim(txtNewModuleName.Text),
        _havePickedId: !IsBlank(varSelectedModuleId),
        _existingRow: If(
            Len(Trim(txtNewModuleName.Text)) > 0,
            LookUp(Skill_Matrix_Modules, Title = Trim(txtNewModuleName.Text)),
            Blank()
        )
    },
    // Decide module id
    Set(
        varModuleIdSafe,
        If(
            _havePickedId,
            // already have an ID from picker
            Text(varSelectedModuleId),
            // else use existing by name or create a new one
            If(
                !IsBlank(_existingRow),
                Text(_existingRow.Module_ID),
                With(
                    { _newId: GUID() },
                    Patch(
                        Skill_Matrix_Modules,
                        Defaults(Skill_Matrix_Modules),
                        {
                            Module_ID: _newId,
                            Title: _typedName,
                            Created_By: User().Email,
                            Created_At: Now()
                        }
                    );
                    Text(_newId)
                )
            )
        )
    );
    // Ensure we have a friendly name cached too
    If(
        IsBlank(varSelectedModuleName),
        Set(varSelectedModuleName, Coalesce(_typedName, varSelectedModuleName))
    )
)
