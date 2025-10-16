        // 6) OPTIONAL: Keep existing Reference rows' Skill_Type synced with CategoryItems
        Clear(colDidUpdate);
        ForAll(
            Filter(Skill_Matrix_Reference As r, Mod_ID = moduleId),
            With(
                { ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = r.CatItem_ID) },
                If(
                    !IsBlank(ci) && ci.Skill_Type <> r.Skill_Type,
                    Patch(Skill_Matrix_Reference, r, { Skill_Type: ci.Skill_Type });
                    Collect(colDidUpdate, { id: r.CatItem_ID })
                )
            )
        );

        // 7) Notify with counts (handle no-op gracefully)
        If(
            CountRows(colDidAdd) + CountRows(colDidRemove) + CountRows(colDidUpdate) = 0,
            Notify("No changes to save.", NotificationType.Information),
            Notify(
                "Saved. Added: " & CountRows(colDidAdd) &
                " | Removed: " & CountRows(colDidRemove) &
                " | Updated types: " & CountRows(colDidUpdate),
                NotificationType.Success
            )
        );

