
## IndexConsumer ##

Helpers:
* GetFileInfo

Has Visitors (listing for compact is-this-already-covered checks):
* VisitToken
* VisitNamedDecl: fancy, covers all of the following with "def"/"decl" smarts:
  * FunctionDecl, w/template instantiation piercing, prettyKing="function"
  * TagDecl: prettyKind="type"
  * TypedefNameDecl: prettyKind="type"
  * VarDecl: ignores local vars/parameters, prettyKind="variable"
  * NamespaceDecl, NamespaceAliasDecl: prettyKind="namespace"
  * FieldDecl: prettyKind="field"
  * EnumConstantDecl: prettyKind="enum constant"
  * (else bail)
  * CXXMethodDecl: invokes FindOverriddenMethods to augment symbols vector
  * CXXDestructorDecl: skips over '~' and whitespace to get to class name.
* VisitCXXConstructExpr: "use" "constructor", pierces template instantiation
* VisitCallExpr
