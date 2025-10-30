@if(
  equals(length(outputs('CMP_AddedRef_Filtered')), 0),
  'Reference_ID eq '''' and IsActive eq 1',
  concat(
    '(',
    join(
      array(
        concat('Reference_ID eq ''', first(outputs('CMP_AddedRef_Filtered')), ''''),
        join(
          skip(outputs('CMP_AddedRef_Filtered'), 1),
          ''' or Reference_ID eq '''
        )
      ),
      ''
    ),
    ') and IsActive eq 1'
  )
)
