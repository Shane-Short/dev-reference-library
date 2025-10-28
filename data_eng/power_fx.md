// 1.1) Resolve or create Module (Option A) — safe scoping
If(
    varHasModuleContext,
    With(
        { tn: Trim(txtNewModuleName.Text) },   // <— capture typed name once
        With(
            {
                existingMod:
                    If(
                        !IsBlank(varSelectedModuleId),
                        LookUp(Skill_Matrix_Modules, Module_ID = varSelectedModuleId),
                        LookUp(Skill_Matrix_Modules, Title = tn)
                    )
            },
            If(
                !IsBlank(existingMod),
                // use existing
                Set(varModuleIdSafe, "" & existingMod.Module_ID);
                If(IsBlank(varSelectedModuleName), Set(varSelectedModuleName, existingMod.Title)),
                // create new
                With(
                    { newId: GUID() },
                    Patch(
                        Skill_Matrix_Modules,
                        Defaults(Skill_Matrix_Modules),
                        {
                            Module_ID: "" & newId,
                            Title: tn,
                            Created_By: reqUserEmail,
                            Created_At: Now()
                        }
                    );
                    Set(varModuleIdSafe, "" & newId);
                    Set(varSelectedModuleName, tn)
                )
            )
        )
    )
)
