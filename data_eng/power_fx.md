@if(
  equals(variables('varEmployeeEmail'), ''),
  outputs('SEL_DeleteModule_Emails'),
  union(
    outputs('SEL_DeleteModule_Emails'),
    createArray(
      json(
        concat(
          '{ "Email": "',
          variables('varEmployeeEmail'),
          '" }'
        )
      )
    )
  )
)
