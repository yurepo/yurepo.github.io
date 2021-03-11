---
title: Parsing A Boolean Expression
tags:
  - JVM
  - Kotlin
  - LeetCode
categories:
  - LeetCode
abbrlink: 41284
date: 2020-08-03 22:02:41
---
題目要求是要**輸入一個運算式回傳該運算式的結果**
有下列規則
+ "t"，代表為 `True`
+ "f"，代表為 `False`
+ "!(expr)"，表示將`expr`得出的布林值反向
+ "&(expr1,expr2,...)"，表示將所有expr{num}做AND
+ "|(expr1,expr2,...)"，表示將所有expr{num}做OR

馬上進入程式碼撰寫的部分，主要是以遞迴解決
我的想法應該算蠻差的，效能那些部分都不能強求，所以僅放上來做參考，歡迎底下討論

```Kotlin
class Solution {

    companion object{
        // 運算子列表
        val operator = arrayOf('!', '&', '|')
    }

    fun parseBoolExpr(expression: String): Boolean {
        // 如果運算式為空，回傳 `False`
        if (expression.isEmpty())
            return false
        return expr(expression)
    }

    fun expr(expression: String): Boolean{
        // 如果運算式的首個元素不包含於運算子列表(!、&、|)，回傳 `False`
        if (!operator.contains(expression[0]))
            return false
        
        // 取得運算子
        val mod = expression[0]
        // 檢查運算子後的下一個字元是否為 `(`
        val optStart = if(expression[1] == '(') 1 else throw RuntimeException("not valid pattern")
        // 檢查運算式的最後一個字源是否為 `)`
        val optEnd = if(expression[expression.lastIndex] == ')') expression.lastIndex else throw RuntimeException("not valid pattern")
        
        val arg = arrayListOf<Boolean>()
        var index = optStart
        while(index <= optEnd){
            // 如果需要遞迴解決
            if (operator.contains(expression[index])){
                val segStartIndex = index + 1
                // 尋找括弧的結束點
                val segEndIndex = findPair(expression, segStartIndex)
                // 遞迴解決並將結果新增到列表中
                arg.add(expr(expression.substring(index, segEndIndex)))
                // 運算完成後略過已經運算完成的運算式
                index = segEndIndex + 1
            }
            else{
                //不需要遞迴解決的話就直接新增到列表中
                when (expression[index]){
                    't' -> arg.add(true)
                    'f' -> arg.add(false)
                }
                index++
            }
        }
        return calculartor(mod, arg).apply {
            println("matching $expression result: $this")
        }
    }

    fun findPair(expr: String, startIndex: Int): Int{
        var pass = 0
        expr.substring(startIndex).forEachIndexed { index, item ->
            when(item){
                '(' -> pass++
                ')' -> pass--
            }
            if (item == ')' && pass == 0){
                return index + startIndex + 1
            }
        }
        return -1
    }

    fun calculartor(mod: Char, list: List<Boolean>): Boolean {
        when(mod){
            '!' -> {
                return !list[0]
            }
            '&' -> {
                return !list.contains(false)
            }
            '|' -> {
                return list.contains(true)
            }
        }
        return false
    }
}
```