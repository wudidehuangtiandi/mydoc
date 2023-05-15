# Solidity语言学习

## 1.基本类型，函数，结构体，array, mapping及修饰词

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//contract 类似其它语言的class
contract SimpleStorage {
    // boolean,uint（只能正）,int（可正负）,address（地址）,bytes（最多32）
    
    // bool hasFavoriteNumber = true;
    // public 关键词会自带getter函数，所以可以被外部看见(retrieve实现可以省略public),这些变量在同一作用域会被自动编号比如favoriteNumber是0
    // uint256 public favoriteNumber;
    // 256 这种可以自定义默认256
    uint256 favoriteNumber;
    // 调用结构体,定义people,内部需要json
    // People public person = People({favoriteNumber:2,name:"gc"});

    //映射，代表每个string 匹配一个uint256值
    mapping(string => uint256) public nameToFavouriteNumber;

    //结构体
    struct People {
        uint256 favoriteNumber;
        string name;
    }

    //array
    People[] public people;

    // string favoriteNumberInText = "Five";
    // int256 favoriteInt = -5;
    // address myAddress = 0xdEb1A63D95970C43f9C3B18DAfDE0b1BF58015dc;
    // bytes32 favoriteBytes = "cat";

    function store(uint256 _fa) public {
        favoriteNumber = _fa;
    }

    //这两个关键词修饰的函数不用花gas,但是要注意，如果是需要消耗gas的方法调用了这两个方法,那么它将会需要支付这两个方法的GAS
    //view(只能读取不能修改),pure(不能读取也不能修改)
    function retrieve() public view returns (uint256) {
        return favoriteNumber;
    }

    function add() public pure returns (uint256) {
        return (1 + 1);
    }

    //关键词calldata, memory, storage。前两个表示暂时存在的。storage是存储变量（代表实际上有地方存储的），上面的favoriteNumber实际上就有隐藏的这个关键词修饰
    //calldata 和memory 的区别类似es6中的const和let。calldata修饰的变量不可变更，另外他们只能用来修饰array struct或者mapping 其他类型不需要
    function addPerson(string memory _name,uint256 _favNumber) public {
        People memory newPerson = People({favoriteNumber:_favNumber,name:_name});
        people.push(newPerson);
        nameToFavouriteNumber[_name]=_favNumber;
    }

  
}
```

