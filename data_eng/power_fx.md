// 6.2 Add what's missing (by CatItem_ID not present in BEFORE)
ClearCollect(
    colRefsToAdd,
    ForAll(
        colTargetCatItems As t,
        With(
            {
                // did we ALREADY have this CatItem_ID active before save?
                existingActiveRef:
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varModuleIdSafe &&
                        CatItem_ID = t.CatItem_ID &&
                        IsActive = true
                    ),

                // did we have it before, but it was soft-deleted?
                existingInactiveRef:
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = varModuleIdSafe &&
                        CatItem_ID = t.CatItem_ID &&
                        IsActive = false
                    ),

                ci:
                    LookUp(
                        colCI_ById,
                        CatItem_ID = t.CatItem_ID
                    ),

                resurrectNow: Now()
            },

            /* We will return a tiny record for diff calc later, but we’ll
               only actually Patch when needed. There are 3 cases:

               1. existingActiveRef found  -> already active, nothing to do.
               2. existingInactiveRef found -> PATCH to reactivate.
               3. neither found            -> INSERT brand new ref row.
            */

            If(
                // Case 1: already active
                !IsBlank(existingActiveRef),

                // just return its Reference_ID for diff tracking
                {
                    Reference_ID: existingActiveRef.Reference_ID,
                    CatItem_ID: t.CatItem_ID
                },

                // Else Case 2 or 3
                If(
                    // Case 2: reactivate inactive row
                    !IsBlank(existingInactiveRef),

                    // PATCH the inactive row to flip it on
                    With(
                        {
                            _patched:
                                Patch(
                                    Skill_Matrix_Reference,
                                    existingInactiveRef,
                                    {
                                        IsActive: true,
                                        // keep the same Module/Mod_ID/Category/etc. as-is
                                        Updated_By: reqUserEmail,
                                        Updated_At: resurrectNow
                                    }
                                )
                        },
                        {
                            Reference_ID: _patched.Reference_ID,
                            CatItem_ID: t.CatItem_ID
                        }
                    ),

                    // Case 3: brand new row
                    With(
                        {
                            newRefId: GUID(),
                            _inserted:
                                Patch(
                                    Skill_Matrix_Reference,
                                    Defaults(Skill_Matrix_Reference),
                                    {
                                        Reference_ID: "" & newRefId,
                                        Module: varSelectedModuleName,
                                        Mod_ID: varModuleIdSafe,
                                        IsActive: true,
                                        CatItem_ID: t.CatItem_ID,
                                        Category: ci.Category,
                                        Item: ci.Item,
                                        Skill_Type: ci.Skill_Type,
                                        Created_By: reqUserEmail,
                                        Created_At: Now()
                                    }
                                )
                        },
                        {
                            Reference_ID: _inserted.Reference_ID,
                            CatItem_ID: t.CatItem_ID
                        }
                    )
                )
            )
        )
    )
);








