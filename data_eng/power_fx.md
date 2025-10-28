// 1.1 Resolve Module_ID (use selected, or create/find by typed name)
With(
    { tName: Trim(txtNewModuleName.Text) },
    With(
        { exMod: LookUp(Skill_Matrix_Modules, Title = tName) },
        Set(
            varModuleIdSafe,
            If(
                !IsBlank(varSelectedModuleId),
                "" & varSelectedModuleId,
                If(
                    !IsBlank(exMod),
                    "" & exMod.Module_ID,
                    With(
                        { newModId: GUID() },
                        Patch(
                            Skill_Matrix_Modules,
                            Defaults(Skill_Matrix_Modules),
                            {
                                Module_ID: newModId,
                                Title: tName,
                                Created_By: User().Email,
                                Created_At: Now()
                            }
                        );
                        "" & newModId
                    )
                )
            )
        );
        If(IsBlank(varSelectedModuleName), Set(varSelectedModuleName, Coalesce(txtNewModuleName.Text, varSelectedModuleName)))
    )
);
