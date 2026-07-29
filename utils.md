## Utilities
1. How do you print a stack trace to the log?
```
Header file
#include <Common/StackTrace.h>

Call: StackTrace().toString()
```
## Logging
1. Print `ColumnsDescription`
```
String ColumnsDescription::toString(bool include_comments) const
{
    WriteBufferFromOwnString buf;
    IAST::FormatState ast_format_state;

    writeCString("columns format version: 1\n", buf);
    DB::writeText(columns.size(), buf);
    writeCString(" columns:\n", buf);

    for (const ColumnDescription & column : columns)
        column.writeText(buf, ast_format_state, include_comments);

    return buf.str();
}
```
2. Print `NameAndTypePair`
```
String NameAndTypePair::dump() const
{
    WriteBufferFromOwnString out;
    out << "name: " << name << "\n"
        << "type: " << type->getName() << "\n"
        << "name in storage: " << getNameInStorage() << "\n"
        << "type in storage: " << getTypeInStorage()->getName();

    return out.str();
}
```
3. Print `ColumnWithTypeAndName`
```
String ColumnWithTypeAndName::dumpStructure() const
{
    WriteBufferFromOwnString out;
    dumpStructure(out);
    return out.str();
}
```
4.   `NamesAndTypesList`
```
void NamesAndTypesList::writeText(WriteBuffer & buf) const
{
    writeString("columns format version: 1\n", buf);
    DB::writeText(size(), buf);
    writeString(" columns:\n", buf);
    for (const auto & it : *this)
    {
        writeBackQuotedString(it.name, buf);
        writeChar(' ', buf);
        writeString(it.type->getName(), buf);
        writeChar('\n', buf);
    }
}

String NamesAndTypesList::toString() const
{
    WriteBufferFromOwnString out;
    writeText(out);
    return out.str();
}
```
5. `MergeTreeReadTaskColumns`
```
String MergeTreeReadTaskColumns::dump() const
{
    WriteBufferFromOwnString s;
    for (size_t i = 0; i < pre_columns.size(); ++i)
    {
        s << "STEP " << i << ": " << pre_columns[i].toString() << "\n";
    }
    s << "COLUMNS: " << columns.toString() << "\n";
    return s.str();
}
```
6. `ActionsDAG` (related to expressions)
```
std::string ActionsDAG::dumpDAG() const
{
    std::unordered_map<const Node *, size_t> map;
    for (const auto & node : nodes)
    {
        size_t idx = map.size();
        map[&node] = idx;
    }

    WriteBufferFromOwnString out;
    for (const auto & node : nodes)
    {
        out << map[&node] << " : ";
        switch (node.type)
        {
            case ActionsDAG::ActionType::COLUMN:
                out << "COLUMN ";
                break;

            case ActionsDAG::ActionType::ALIAS:
                out << "ALIAS ";
                break;

            case ActionsDAG::ActionType::FUNCTION:
                out << "FUNCTION ";
                break;

            case ActionsDAG::ActionType::ARRAY_JOIN:
                out << "ARRAY JOIN ";
                break;

            case ActionsDAG::ActionType::INPUT:
                out << "INPUT ";
                break;
        }

        out << "(";
        for (size_t i = 0; i < node.children.size(); ++i)
        {
            if (i)
                out << ", ";
            out << map[node.children[i]];
        }
        out << ")";

        out << " " << (node.column ? node.column->getName() : "(no column)");
        out << " " << (node.result_type ? node.result_type->getName() : "(no type)");
        out << " " << (!node.result_name.empty() ? node.result_name : "(no name)");

        if (node.function_base)
            out << " [" << node.function_base->getName() << "]";

        if (node.is_function_compiled)
            out << " [compiled]";

        out << "\n";
    }

    out << "Output nodes:";
    for (const auto * node : outputs)
        out << ' ' << map[node];
    out << '\n';

    return out.str();
}
```
dumpNames:
```
std::string ActionsDAG::dumpNames() const
{
    WriteBufferFromOwnString out;
    for (auto it = nodes.begin(); it != nodes.end(); ++it)
    {
        if (it != nodes.begin())
            out << ", ";
        out << it->result_name;
    }
    return out.str();
}
```
7. `ExpressionActions`
```
std::string ExpressionActions::dumpActions() const
{
    WriteBufferFromOwnString ss;

    ss << "input:\n";
    for (const auto & input_column : required_columns)
        ss << input_column.name << " " << input_column.type->getName() << "\n";

    ss << "\nactions:\n";
    for (const auto & action : actions)
        ss << action.toString() << '\n';

    ss << "\noutput:\n";
    NamesAndTypesList output_columns = sample_block.getNamesAndTypesList();
    for (const auto & output_column : output_columns)
        ss << output_column.name << " " << output_column.type->getName() << "\n";

    ss << "\noutput positions:";
    for (auto pos : result_positions)
        ss << " " << pos;
    ss << "\n";

    return ss.str();
}
```
8. Print all data in a Block
```
# Block additional_columns;

for (size_t i = 0; i < additional_columns.columns(); ++i)
{
    auto column = additional_columns.getByPosition(i);
    for (size_t j = 0; j < column.column->size(); ++j)
    {
        LOG_DEBUG(getLogger("additional_columns"), "current column:{},index:{},value:{}",
                            column.name, j, toString((*column.column)[j]));
    }
}
```
