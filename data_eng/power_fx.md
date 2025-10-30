concat(
  'Employee_Email eq ''',
    toLower(items('EACH_User')?['Email']),
  ''' and Category eq ''',
    items('EACH_Ref')?['Category'],
  ''' and Item eq ''',
    items('EACH_Ref')?['Item'],
  ''' and IsActive eq 1'
)
