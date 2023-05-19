# Solidity语言学习

> 我们先使用BS编译器remix https://remix.ethereum.org/ 来作以下演示。

## 1.基本类型，函数，结构体，array, mapping及修饰词

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;
// SPDX这个一定要有

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

## 2.和其它合约交互

```solidity
// SPDX-License-Identifier: MIT 

pragma solidity ^0.8.7;

import "./SimpleStorage.sol"; 

//本节展示如何和其他合约交互，假设SimpleStorage是另一个合约
contract StorageFactory {
    
    SimpleStorage[] public simpleStorageArray;
    
    //创建这个合约并且丢到数组里去
    function createSimpleStorageContract() public {
        SimpleStorage simpleStorage = new SimpleStorage();
        simpleStorageArray.push(simpleStorage);
    }
    
    //给指定index的合约赋予一个喜欢的数字
    function sfStore(uint256 _simpleStorageIndex, uint256 _simpleStorageNumber) public {
        // Address 
        // ABI 
        // SimpleStorage(address(simpleStorageArray[_simpleStorageIndex])).store(_simpleStorageNumber);
        simpleStorageArray[_simpleStorageIndex].store(_simpleStorageNumber);
    }
    
    //查看指定index的合约喜欢的数字
    function sfGet(uint256 _simpleStorageIndex) public view returns (uint256) {
        // return SimpleStorage(address(simpleStorageArray[_simpleStorageIndex])).retrieve();
        return simpleStorageArray[_simpleStorageIndex].retrieve();
    }
}
```

### 2.1继承和重写

```solidity

function store(uint256 _favoriteNumber) public virtual {
    favoriteNumber = _favoriteNumber;
}
```



```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

//继承,分两步先import再 is下

import "./SimpleStorage.sol";

//重写方法，被重写的方法必须加上virtual 关键词允许重写，重写的方法要加上override关键词
contract ExtraStorage is SimpleStorage {
    function store(uint256 _favoriteNumber) public override  {
        favoriteNumber = _favoriteNumber + 5;
    }
}

```

## 3.预言机的初步使用

> 合约在与外界交互时通过可信的区中心化的预言机来实现。我们这采用chainlink，包括价格获取，API调用等等服务
>
> 下面的demo演示如何使用chainlink来判断发送的eth是否价值50美元，需要实时获取价格和计算传入eth的价值。顺便将发送的代币和其地址存到合约的address mapping中

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//预言机地址https://docs.chain.link/data-feeds/using-data-feeds

//从github仓库拉取接口下面这个地址是从预言机的文档中抄的,
//仓库合约地址：https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract FundMe {
    uint256 public minimumUsd = 50;

    address[] public funders;

    mapping (address => uint256) public addressToAmountFunded;

    // 关键词payable，使得函数变红(意思是这个函数可以发送一些代币)
    function fund() public payable {
        //关键词msg.value 获取发送的value,默认单位wei，十的十八位
        //关键词require，满足则通过不满足则返回第二个参数
        //require (msg.value > 1e18, "Did not send enough");

        require(getConversionRate(msg.value) > minimumUsd, "Did not send enough");

        //msg.sender 为关键词，记录调用人地址
        funders.push(msg.sender);
        //记录某某人发了多少
        addressToAmountFunded[msg.sender]=msg.value;
    }

    //获取eth对美元的价格
    function getPrice() public view returns (uint256) {
        //ABI可以从文档或者github看到
        //预言机中获取eth对美元的ADDRESS 0x694AA1769357215DE4FAC081bf1f309aDC325306

        AggregatorV3Interface priceFeed = AggregatorV3Interface(
            0x694AA1769357215DE4FAC081bf1f309aDC325306
        );
        //这个依然是看文档获得的返回值，或者合约源码也有
        (, int256 price, , , ) = priceFeed.latestRoundData();
        // answer有八位在小数之后，solidity无法表示小数，也可以调用合约AggregatorV3Interface方法decimals来得到
        // 这就涉及到了单位转换，我们要把返回值乘以十的十次方，才能得到和我们value相同的位数,然后需要将int转换为uint类型。
        return uint256(price * 1e10);
    }

    //计算传入的eth值多少美元
    function getConversionRate(uint256 ethAmount)
        public
        view
        returns (uint256)
    {
        uint256 ethPrice = getPrice();
        uint256 ethAmountInUsd = (ethPrice * ethAmount) / 1e18;

        return ethAmountInUsd;
    }
}

```

## 4.库

> 可以用来增强某些类型功能

### 4.1基本方式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

//库里的方法必须时internal修饰
library PriceConverter {
      //获取eth对美元的价格
    function getPrice() internal view returns (uint256) {
        //ABI可以从文档或者github看到
        //预言机中获取eth对美元的ADDRESS 0x694AA1769357215DE4FAC081bf1f309aDC325306

        AggregatorV3Interface priceFeed = AggregatorV3Interface(
            0x694AA1769357215DE4FAC081bf1f309aDC325306
        );
        //这个依然是看文档获得的返回值，或者合约源码也有
        (, int256 price, , , ) = priceFeed.latestRoundData();
        // answer有八位在小数之后，solidity无法表示小数，也可以调用合约AggregatorV3Interface方法decimals来得到
        // 这就涉及到了单位转换，我们要把返回值乘以十的十次方，才能得到和我们value相同的位数,然后需要将int转换为uint类型。
        return uint256(price * 1e10);
    }

    //计算传入的eth值多少美元
    function getConversionRate(uint256 ethAmount)
        internal
        view
        returns (uint256)
    {
        uint256 ethPrice = getPrice();
        uint256 ethAmountInUsd = (ethPrice * ethAmount) / 1e18;

        return ethAmountInUsd;
    }
}
```

> 调用，先引入，再for一下，最后才能使用。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//引入
import "./PriceConverter.sol";

contract FundMe {
    //给uint256类型增加Library
    using PriceConverter for uint256;

    uint256 public minimumUsd = 50;

    address[] public funders;

    mapping(address => uint256) public addressToAmountFunded;

    function fund() public payable {
        //msg.value会被当作第一个参数传入，如果有第二个参数需要放在括号里
        require(
            msg.value.getConversionRate() > minimumUsd,
            "Did not send enough"
        );

        funders.push(msg.sender);

        addressToAmountFunded[msg.sender] = msg.value;
    }
}
```

## 5.uncheked关键词

> 8.0以后可以用来加在计算之后，使得超过上限时头从开始。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

contract SafeMathTester {
    uint8 public bigNumber = 255;

    function add() public {
    //8.0之后的版本不加就会保持再255上限
        unchecked {
            bigNumber = bigNumber + 1;
        }
    }
}
```

## 6.取款

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//引入
import "./PriceConverter.sol";

contract FundMe {
    //给uint256类型增加Library
    using PriceConverter for uint256;

    uint256 public minimumUsd = 50;

    address[] public funders;

    mapping(address => uint256) public addressToAmountFunded;

    function fund() public payable {
        //msg.value会被当作第一个参数传入，如果有第二个参数需要放在括号里
        require(
            msg.value.getConversionRate() > minimumUsd,
            "Did not send enough"
        );

        funders.push(msg.sender);

        addressToAmountFunded[msg.sender] = msg.value;
    }

    //取钱，目的将合约中的钱发给调用者
    function withdraw() public {
        //数组循环
        // for(uint256 funderIndex =0; funderIndex< funders.length; funderIndex =funderIndex +1){
        //     address funder = funders[funderIndex];
        //     addressToAmountFunded[funder] = 0;
        // }
        //重置
        //funders = new address[](0);
           
        //下面是三种提款方式，payable(msg.sender) 把msg.sender从address类型转换成payable address类型，address(this).balance表示调用者合约本身的存款余额
        //transfer(最多2300gas,失败会直接报错)
        //payable(msg.sender).transfer(address(this).balance);

        //send(最多2300gas失败返回一个布尔)
        //bool sendSucess = payable(msg.sender).send(address(this).balance);
        //加上require,失败会回滚交易
        //require(sendSuccess,"Send failed");

        //call(没有gas上限，返回bool，推荐)
        (bool callSuccess,)=payable(msg.sender).call{value: address(this).balance}("");
        require(callSuccess,"Call failed");
    }
}
```

## 7.构造函数和修饰器

> 现在我们只想这个取钱函数被拥有者调用

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//引入
import "./PriceConverter.sol";

contract FundMe {
    //给uint256类型增加Library
    using PriceConverter for uint256;

    uint256 public minimumUsd = 50;

    address[] public funders;

    mapping(address => uint256) public addressToAmountFunded;

    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function fund() public payable {
        //msg.value会被当作第一个参数传入，如果有第二个参数需要放在括号里
        require(
            msg.value.getConversionRate() > minimumUsd,
            "Did not send enough"
        );

        funders.push(msg.sender);

        addressToAmountFunded[msg.sender] = msg.value;
    }

    //取钱，目的将合约中的钱发给调用者
    function withdraw() public onlyOwner{
        // 单个方式
        // require(msg.sender == owner, "not owner");

        //数组循环
        // for(uint256 funderIndex =0; funderIndex< funders.length; funderIndex =funderIndex +1){
        //     address funder = funders[funderIndex];
        //     addressToAmountFunded[funder] = 0;
        // }
        //重置
        //funders = new address[](0);

        //下面是三种提款方式，payable(msg.sender) 把msg.sender从address类型转换成payable address类型，address(this).balance表示调用者合约本身的存款余额
        //transfer(最多2300gas,失败会直接报错)
        //payable(msg.sender).transfer(address(this).balance);

        //send(最多2300gas失败返回一个布尔)
        //bool sendSucess = payable(msg.sender).send(address(this).balance);
        //加上require,失败会回滚交易
        //require(sendSuccess,"Send failed");

        //call(没有gas上限，返回bool，推荐)
        (bool callSuccess, ) = payable(msg.sender).call{
            value: address(this).balance
        }("");
        require(callSuccess, "Call failed");
    }

    //修饰器，方法后面加上这个，会先执行这个，下划线表示加上修饰器的方法和修饰器执行的顺序
    //像这样表示先执行修饰器再执行方法，如果下划线在上面表示先执行方法再执行修饰器。
    modifier onlyOwner {
        require(msg.sender == owner, "not owner");
        _;
    }
}
```

## 8.Immutable和Constant

> 常量使用constant，只可操作一次的变量使用immutable 将会明显减少gas消耗

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//引入
import "./PriceConverter.sol";

contract FundMe {
    //给uint256类型增加Library
    using PriceConverter for uint256;

    //constant 能减少GAS消耗
    uint256 public constant MINIMUM_USD = 50;

    address[] public funders;

    mapping(address => uint256) public addressToAmountFunded;

    //只可编译一次
    address public immutable owner;

    constructor() {
        owner = msg.sender;
    }

    function fund() public payable {
        //msg.value会被当作第一个参数传入，如果有第二个参数需要放在括号里
        require(
            msg.value.getConversionRate() > MINIMUM_USD,
            "Did not send enough"
        );

        funders.push(msg.sender);

        addressToAmountFunded[msg.sender] = msg.value;
    }

    //取钱，目的将合约中的钱发给调用者
    function withdraw() public onlyOwner{
        // 单个方式
        // require(msg.sender == owner, "not owner");

        //数组循环
        // for(uint256 funderIndex =0; funderIndex< funders.length; funderIndex =funderIndex +1){
        //     address funder = funders[funderIndex];
        //     addressToAmountFunded[funder] = 0;
        // }
        //重置
        //funders = new address[](0);

        //下面是三种提款方式，payable(msg.sender) 把msg.sender从address类型转换成payable address类型，address(this).balance表示调用者合约本身的存款余额
        //transfer(最多2300gas,失败会直接报错)
        //payable(msg.sender).transfer(address(this).balance);

        //send(最多2300gas失败返回一个布尔)
        //bool sendSucess = payable(msg.sender).send(address(this).balance);
        //加上require,失败会回滚交易
        //require(sendSuccess,"Send failed");

        //call(没有gas上限，返回bool，推荐)
        (bool callSuccess, ) = payable(msg.sender).call{
            value: address(this).balance
        }("");
        require(callSuccess, "Call failed");
    }

    //修饰器，方法后面加上这个，会先执行这个，下划线表示加上修饰器的方法和修饰器执行的顺序
    //像这样表示先执行修饰器再执行方法，如果下划线在上面表示先执行方法再执行修饰器。
    modifier onlyOwner {
        require(msg.sender == owner, "not owner");
        _;
    }

}
```

## 9.自定义错误

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//引入
import "./PriceConverter.sol";

//定义错误
error NotOwner();

contract FundMe {
    //给uint256类型增加Library
    using PriceConverter for uint256;

    //constant 能减少GAS消耗
    uint256 public constant MINIMUM_USD = 50;

    address[] public funders;

    mapping(address => uint256) public addressToAmountFunded;

    address public immutable owner;

    constructor() {
        owner = msg.sender;
    }

    function fund() public payable {
        //msg.value会被当作第一个参数传入，如果有第二个参数需要放在括号里
        require(
            msg.value.getConversionRate() > MINIMUM_USD,
            "Did not send enough"
        );

        funders.push(msg.sender);

        addressToAmountFunded[msg.sender] = msg.value;
    }

    //取钱，目的将合约中的钱发给调用者
    function withdraw() public onlyOwner{
        // 单个方式
        // require(msg.sender == owner, "not owner");

        //数组循环
        // for(uint256 funderIndex =0; funderIndex< funders.length; funderIndex =funderIndex +1){
        //     address funder = funders[funderIndex];
        //     addressToAmountFunded[funder] = 0;
        // }
        //重置
        //funders = new address[](0);

        //下面是三种提款方式，payable(msg.sender) 把msg.sender从address类型转换成payable address类型，address(this).balance表示调用者合约本身的存款余额
        //transfer(最多2300gas,失败会直接报错)
        //payable(msg.sender).transfer(address(this).balance);

        //send(最多2300gas失败返回一个布尔)
        //bool sendSucess = payable(msg.sender).send(address(this).balance);
        //加上require,失败会回滚交易
        //require(sendSuccess,"Send failed");

        //call(没有gas上限，返回bool，推荐)
        (bool callSuccess, ) = payable(msg.sender).call{
            value: address(this).balance
        }("");
        require(callSuccess, "Call failed");
    }

    //修饰器，方法后面加上这个，会先执行这个，下划线表示加上修饰器的方法和修饰器执行的顺序
    //像这样表示先执行修饰器再执行方法，如果下划线在上面表示先执行方法再执行修饰器。
    modifier onlyOwner {
        //require(msg.sender == owner, "not owner");
        //使用if revert能节省大量GAS
        if(msg.sender != owner){revert NotOwner();}
        _;
    }
}
```

## 10.特殊方法

> 比如某些人没使用上面定义的fund方法，像合约发送eth，这时候可以使用特殊方法来做处理

当向合约转账却没有调用正确的方法时会触发receive，当调用方法未找到时会调用fallback

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

contract FallbackExample {
    uint256 public result;

//特殊函数省略function ,关键词payable标红，external是文档要求的关键词
    receive() external payable {
        result = 1;
    }
    fallback() external payable{
        result = 2;
    }
}
```

修改我们的合约，以处理哪些没正确调用存款方法的转账。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

//引入
import "./PriceConverter.sol";

//定义错误
error NotOwner();

contract FundMe {
    //给uint256类型增加Library
    using PriceConverter for uint256;

    //constant 能减少GAS消耗
    uint256 public constant MINIMUM_USD = 50;

    address[] public funders;

    mapping(address => uint256) public addressToAmountFunded;

    address public immutable owner;

    constructor() {
        owner = msg.sender;
    }

    function fund() public payable {
        //msg.value会被当作第一个参数传入，如果有第二个参数需要放在括号里
        require(
            msg.value.getConversionRate() > MINIMUM_USD,
            "Did not send enough"
        );

        funders.push(msg.sender);

        addressToAmountFunded[msg.sender] = msg.value;
    }

    //取钱，目的将合约中的钱发给调用者
    function withdraw() public onlyOwner {
        // 单个方式
        // require(msg.sender == owner, "not owner");

        //数组循环
        // for(uint256 funderIndex =0; funderIndex< funders.length; funderIndex =funderIndex +1){
        //     address funder = funders[funderIndex];
        //     addressToAmountFunded[funder] = 0;
        // }
        //重置
        //funders = new address[](0);

        //下面是三种提款方式，payable(msg.sender) 把msg.sender从address类型转换成payable address类型，address(this).balance表示调用者合约本身的存款余额
        //transfer(最多2300gas,失败会直接报错)
        //payable(msg.sender).transfer(address(this).balance);

        //send(最多2300gas失败返回一个布尔)
        //bool sendSucess = payable(msg.sender).send(address(this).balance);
        //加上require,失败会回滚交易
        //require(sendSuccess,"Send failed");

        //call(没有gas上限，返回bool，推荐)
        (bool callSuccess, ) = payable(msg.sender).call{
            value: address(this).balance
        }("");
        require(callSuccess, "Call failed");
    }

    //修饰器，方法后面加上这个，会先执行这个，下划线表示加上修饰器的方法和修饰器执行的顺序
    //像这样表示先执行修饰器再执行方法，如果下划线在上面表示先执行方法再执行修饰器。
    modifier onlyOwner() {
        //require(msg.sender == owner, "not owner");
        if (msg.sender != owner) {
            revert NotOwner();
        }
        _;
    }

    receive() external payable {
        fund();
    }

    fallback() external payable {
        fund();
    }
}
```

!>至此我们结束了remix上的演示，下面将转入vscode环境

## 11.本地环境

需要模拟的区块链节点：https://github.com/trufflesuite/ganache-ui/releases  去这里下载

需要solcjs来编译sol文件，需要ethers来和合约交互

```
{
  "name": "ethers-simple-storage-fcc",
  "version": "1.0.0",
  "description": "",
  "dependencies": {
    "dotenv": "^14.2.0",
    "ethers": "^6.2.3",
    "prettier": "^2.5.1",
    "solc": "0.8.7-fixed"
  },
  "scripts": {
    "compile": "solcjs --bin --abi --include-path node_modules/ --base-path . -o . SimpleStorage.sol"
  }
}

```

> 钢铁侠爸爸说过，在学会走路前要先学会跑步。我们基础先学到这。后面直接抄起现成的项目学习。