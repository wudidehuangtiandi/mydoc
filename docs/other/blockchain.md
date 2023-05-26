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

## 12.基于bep20的一个template

> 能用，直接抄的开源的，改了个名字，实现了bep20的必要条件，用它直接用remix打包发布了躺平币，bsc链成本很低，2刀，我用它发布了躺平币

`BEP20Token`

```solidity
pragma solidity 0.5.16;

import "./Context.sol";
import "./IBEP20.sol";
import "./Ownable.sol";
import "./SafeMath.sol";

contract BEP20Token is Context, IBEP20, Ownable {
  using SafeMath for uint256;

  mapping (address => uint256) private _balances;

  mapping (address => mapping (address => uint256)) private _allowances;

  uint256 private _totalSupply;
  uint8 private _decimals;
  string private _symbol;
  string private _name;

  constructor() public {
    //名称
    _name = "TangPing";
    //符号
    _symbol = "TPING";
    //精度
    _decimals = 6;
    //总供应量
    _totalSupply = 10000000000000000;
    _balances[msg.sender] = _totalSupply;

    emit Transfer(address(0), msg.sender, _totalSupply);
  }

  /**
   * @dev Returns the bep token owner.
   */
  function getOwner() external view returns (address) {
    return owner();
  }

  /**
   * @dev Returns the token decimals.
   */
  function decimals() external view returns (uint8) {
    return _decimals;
  }

  /**
   * @dev Returns the token symbol.
   */
  function symbol() external view returns (string memory) {
    return _symbol;
  }

  /**
  * @dev Returns the token name.
  */
  function name() external view returns (string memory) {
    return _name;
  }

  /**
   * @dev See {BEP20-totalSupply}.
   */
  function totalSupply() external view returns (uint256) {
    return _totalSupply;
  }

  /**
   * @dev See {BEP20-balanceOf}.//余额
   */
  function balanceOf(address account) external view returns (uint256) {
    return _balances[account];
  }

  /**
   * @dev See {BEP20-transfer}.//转移
   *
   * Requirements:
   * 
   * - `recipient` cannot be the zero address. “收件人”不能是零地址。
   * - the caller must have a balance of at least `amount`.发送人至少要有什么多钱
   */
  function transfer(address recipient, uint256 amount) external returns (bool) {
    _transfer(_msgSender(), recipient, amount);
    return true;
  }

  //解释下下两个方法
  //假设我们有用户A和用户B。A有 1000 个代币，想授权B使用其中的 100 个。
  //一个会打电话approve(address(B), 100, {"from": address(A)})
  //B将检查A允许他使用多少令牌，方法是调用：allowance(address(A), address(B))
  //B将通过调用以下方式将这些令牌发送到他的帐户： transferFrom(address(A), address(B), 100, {"from": address(B)})

  /**
   * @dev See {BEP20-allowance}. //批准spender从owner地址转移代币
   */
  function allowance(address owner, address spender) external view returns (uint256) {
    return _allowances[owner][spender];
  }

  /**
   * @dev See {BEP20-approve}.//批准某地址某数量
   *
   * Requirements:
   *
   * - `spender` cannot be the zero address.
   */
  function approve(address spender, uint256 amount) external returns (bool) {
      //来自_msgSender
    _approve(_msgSender(), spender, amount);
    return true;
  }

  /**
   * @dev See {BEP20-transferFrom}.
   *
   * Emits an {Approval} event indicating the updated allowance. This is not
   * required by the EIP. See the note at the beginning of {BEP20};
   *
   * Requirements:
   * - `sender` and `recipient` cannot be the zero address.
   * - `sender` must have a balance of at least `amount`.
   * - the caller must have allowance for `sender`'s tokens of at least
   * `amount`.
   */
  function transferFrom(address sender, address recipient, uint256 amount) external returns (bool) {
    _transfer(sender, recipient, amount);
    _approve(sender, _msgSender(), _allowances[sender][_msgSender()].sub(amount, "BEP20: transfer amount exceeds allowance"));
    return true;
  }

  /**
   * @dev Atomically increases the allowance granted to `spender` by the caller.
   *
   * This is an alternative to {approve} that can be used as a mitigation for
   * problems described in {BEP20-approve}.
   *
   * Emits an {Approval} event indicating the updated allowance.
   *
   * Requirements:
   *
   * - `spender` cannot be the zero address.
   */
  function increaseAllowance(address spender, uint256 addedValue) public returns (bool) {
    _approve(_msgSender(), spender, _allowances[_msgSender()][spender].add(addedValue));
    return true;
  }

  /**
   * @dev Atomically decreases the allowance granted to `spender` by the caller.
   *
   * This is an alternative to {approve} that can be used as a mitigation for
   * problems described in {BEP20-approve}.
   *
   * Emits an {Approval} event indicating the updated allowance.
   *
   * Requirements:
   *
   * - `spender` cannot be the zero address.
   * - `spender` must have allowance for the caller of at least
   * `subtractedValue`.
   */
  function decreaseAllowance(address spender, uint256 subtractedValue) public returns (bool) {
    _approve(_msgSender(), spender, _allowances[_msgSender()][spender].sub(subtractedValue, "BEP20: decreased allowance below zero"));
    return true;
  }

  /**
   * @dev Creates `amount` tokens and assigns them to `msg.sender`, increasing //持有者增发
   * the total supply.
   *
   * Requirements
   *
   * - `msg.sender` must be the token owner
   */
  function mint(uint256 amount) public onlyOwner returns (bool) {
    _mint(_msgSender(), amount);
    return true;
  }

  /**
   * @dev Moves tokens `amount` from `sender` to `recipient`.
   *
   * This is internal function is equivalent to {transfer}, and can be used to
   * e.g. implement automatic token fees, slashing mechanisms, etc.
   *
   * Emits a {Transfer} event.
   *
   * Requirements:
   *
   * - `sender` cannot be the zero address. //发送人
   * - `recipient` cannot be the zero address. //接收者
   * - `sender` must have a balance of at least `amount`. //转移金额
   */
  function _transfer(address sender, address recipient, uint256 amount) internal {
    require(sender != address(0), "BEP20: transfer from the zero address");
    require(recipient != address(0), "BEP20: transfer to the zero address");

    _balances[sender] = _balances[sender].sub(amount, "BEP20: transfer amount exceeds balance");
    _balances[recipient] = _balances[recipient].add(amount);
    //记录日志
    emit Transfer(sender, recipient, amount);
  }

  /** @dev Creates `amount` tokens and assigns them to `account`, increasing
   * the total supply.//给account地址增发
   *
   * Emits a {Transfer} event with `from` set to the zero address.
   *
   * Requirements
   *
   * - `to` cannot be the zero address.
   */
  function _mint(address account, uint256 amount) internal {
    require(account != address(0), "BEP20: mint to the zero address");

    _totalSupply = _totalSupply.add(amount);
    _balances[account] = _balances[account].add(amount);
    emit Transfer(address(0), account, amount);
  }

  /**
   * @dev Destroys `amount` tokens from `account`, reducing the
   * total supply.//销毁account地址的代币
   *
   * Emits a {Transfer} event with `to` set to the zero address.
   *
   * Requirements
   *
   * - `account` cannot be the zero address.
   * - `account` must have at least `amount` tokens.
   */
  function _burn(address account, uint256 amount) internal {
    require(account != address(0), "BEP20: burn from the zero address");

    _balances[account] = _balances[account].sub(amount, "BEP20: burn amount exceeds balance");
    _totalSupply = _totalSupply.sub(amount);
    emit Transfer(account, address(0), amount);
  }

  /**
   * @dev Sets `amount` as the allowance of `spender` over the `owner`s tokens.//批准
   *
   * This is internal function is equivalent to `approve`, and can be used to
   * e.g. set automatic allowances for certain subsystems, etc.
   *
   * Emits an {Approval} event.
   *
   * Requirements:
   *
   * - `owner` cannot be the zero address.
   * - `spender` cannot be the zero address.
   */
  function _approve(address owner, address spender, uint256 amount) internal {
    require(owner != address(0), "BEP20: approve from the zero address");
    require(spender != address(0), "BEP20: approve to the zero address");

    _allowances[owner][spender] = amount;
    emit Approval(owner, spender, amount);
  }

  /**
   * @dev Destroys `amount` tokens from `account`.`amount` is then deducted 开发者从发来的地址中销毁指定数量的代币
   * from the caller's allowance.
   * 
   * See {_burn} and {_approve}.
   */
  function _burnFrom(address account, uint256 amount) internal {
    _burn(account, amount);
    _approve(account, _msgSender(), _allowances[account][_msgSender()].sub(amount, "BEP20: burn amount exceeds allowance"));
  }
}
```

`Context.sol`

```solidity
pragma solidity 0.5.16;

/*
 * @dev Provides information about the current execution context, including the
 * sender of the transaction and its data. While these are generally available
 * via msg.sender and msg.data, they should not be accessed in such a direct
 * manner, since when dealing with GSN meta-transactions the account sending and
 * paying for execution may not be the actual sender (as far as an application
 * is concerned).
 *
 * This contract is only required for intermediate, library-like contracts.
 */
contract Context {
  // Empty internal constructor, to prevent people from mistakenly deploying
  // an instance of this contract, which should be used via inheritance.
  constructor () internal { }

  function _msgSender() internal view returns (address payable) {
    return msg.sender;
  }

  function _msgData() internal view returns (bytes memory) {
    this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691
    return msg.data;
  }
}
```

`IBEP20.SOL`

```solidity
pragma solidity 0.5.16;

interface IBEP20 {
  /**
   * @dev Returns the amount of tokens in existence.
   */
  function totalSupply() external view returns (uint256);

  /**
   * @dev Returns the token decimals.
   */
  function decimals() external view returns (uint8);

  /**
   * @dev Returns the token symbol.
   */
  function symbol() external view returns (string memory);

  /**
  * @dev Returns the token name.
  */
  function name() external view returns (string memory);

  /**
   * @dev Returns the bep token owner.
   */
  function getOwner() external view returns (address);

  /**
   * @dev Returns the amount of tokens owned by `account`.
   */
  function balanceOf(address account) external view returns (uint256);

  /**
   * @dev Moves `amount` tokens from the caller's account to `recipient`.
   *
   * Returns a boolean value indicating whether the operation succeeded.
   *
   * Emits a {Transfer} event.
   */
  function transfer(address recipient, uint256 amount) external returns (bool);

  /**
   * @dev Returns the remaining number of tokens that `spender` will be
   * allowed to spend on behalf of `owner` through {transferFrom}. This is
   * zero by default.
   *
   * This value changes when {approve} or {transferFrom} are called.
   */
  function allowance(address _owner, address spender) external view returns (uint256);

  /**
   * @dev Sets `amount` as the allowance of `spender` over the caller's tokens.
   *
   * Returns a boolean value indicating whether the operation succeeded.
   *
   * IMPORTANT: Beware that changing an allowance with this method brings the risk
   * that someone may use both the old and the new allowance by unfortunate
   * transaction ordering. One possible solution to mitigate this race
   * condition is to first reduce the spender's allowance to 0 and set the
   * desired value afterwards:
   * https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
   *
   * Emits an {Approval} event.
   */
  function approve(address spender, uint256 amount) external returns (bool);

  /**
   * @dev Moves `amount` tokens from `sender` to `recipient` using the
   * allowance mechanism. `amount` is then deducted from the caller's
   * allowance.
   *
   * Returns a boolean value indicating whether the operation succeeded.
   *
   * Emits a {Transfer} event.
   */
  function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

  /**
   * @dev Emitted when `value` tokens are moved from one account (`from`) to
   * another (`to`).
   *
   * Note that `value` may be zero.
   */
  event Transfer(address indexed from, address indexed to, uint256 value);

  /**
   * @dev Emitted when the allowance of a `spender` for an `owner` is set by
   * a call to {approve}. `value` is the new allowance.
   */
  event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

`Ownable.sol`

```solidity
pragma solidity 0.5.16;

import "./Context.sol";
/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * By default, the owner account will be the one that deploys the contract. This
 * can later be changed with {transferOwnership}.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
contract Ownable is Context {
  address private _owner;

  event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

  /**
   * @dev Initializes the contract setting the deployer as the initial owner.
   */
  constructor () internal {
    address msgSender = _msgSender();
    _owner = msgSender;
    emit OwnershipTransferred(address(0), msgSender);
  }

  /**
   * @dev Returns the address of the current owner.
   */
  function owner() public view returns (address) {
    return _owner;
  }

  /**
   * @dev Throws if called by any account other than the owner.
   */
  modifier onlyOwner() {
    require(_owner == _msgSender(), "Ownable: caller is not the owner");
    _;
  }

  /**
   * @dev Leaves the contract without owner. It will not be possible to call
   * `onlyOwner` functions anymore. Can only be called by the current owner.
   *
   * NOTE: Renouncing ownership will leave the contract without an owner,
   * thereby removing any functionality that is only available to the owner.
   */
  function renounceOwnership() public onlyOwner {
    emit OwnershipTransferred(_owner, address(0));
    _owner = address(0);
  }

  /**
   * @dev Transfers ownership of the contract to a new account (`newOwner`).
   * Can only be called by the current owner.
   */
  function transferOwnership(address newOwner) public onlyOwner {
    _transferOwnership(newOwner);
  }

  /**
   * @dev Transfers ownership of the contract to a new account (`newOwner`).
   */
  function _transferOwnership(address newOwner) internal {
    require(newOwner != address(0), "Ownable: new owner is the zero address");
    emit OwnershipTransferred(_owner, newOwner);
    _owner = newOwner;
  }
}
```

`SafeMath.sol`

```solidity
pragma solidity 0.5.16;

/**
 * @dev Wrappers over Solidity's arithmetic operations with added overflow
 * checks.
 *
 * Arithmetic operations in Solidity wrap on overflow. This can easily result
 * in bugs, because programmers usually assume that an overflow raises an
 * error, which is the standard behavior in high level programming languages.
 * `SafeMath` restores this intuition by reverting the transaction when an
 * operation overflows.
 *
 * Using this library instead of the unchecked operations eliminates an entire
 * class of bugs, so it's recommended to use it always.
 */
library SafeMath {
  /**
   * @dev Returns the addition of two unsigned integers, reverting on
   * overflow.
   *
   * Counterpart to Solidity's `+` operator.
   *
   * Requirements:
   * - Addition cannot overflow.
   */
  function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    require(c >= a, "SafeMath: addition overflow");

    return c;
  }

  /**
   * @dev Returns the subtraction of two unsigned integers, reverting on
   * overflow (when the result is negative).
   *
   * Counterpart to Solidity's `-` operator.
   *
   * Requirements:
   * - Subtraction cannot overflow.
   */
  function sub(uint256 a, uint256 b) internal pure returns (uint256) {
    return sub(a, b, "SafeMath: subtraction overflow");
  }

  /**
   * @dev Returns the subtraction of two unsigned integers, reverting with custom message on
   * overflow (when the result is negative).
   *
   * Counterpart to Solidity's `-` operator.
   *
   * Requirements:
   * - Subtraction cannot overflow.
   */
  function sub(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
    require(b <= a, errorMessage);
    uint256 c = a - b;
    
    return c;
  }

  /**
   * @dev Returns the multiplication of two unsigned integers, reverting on
   * overflow.
   *
   * Counterpart to Solidity's `*` operator.
   *
   * Requirements:
   * - Multiplication cannot overflow.
   */
  function mul(uint256 a, uint256 b) internal pure returns (uint256) {
    // Gas optimization: this is cheaper than requiring 'a' not being zero, but the
    // benefit is lost if 'b' is also tested.
    // See: https://github.com/OpenZeppelin/openzeppelin-contracts/pull/522
    if (a == 0) {
      return 0;
    }

    uint256 c = a * b;
    require(c / a == b, "SafeMath: multiplication overflow");

    return c;
  }

  /**
   * @dev Returns the integer division of two unsigned integers. Reverts on
   * division by zero. The result is rounded towards zero.
   *
   * Counterpart to Solidity's `/` operator. Note: this function uses a
   * `revert` opcode (which leaves remaining gas untouched) while Solidity
   * uses an invalid opcode to revert (consuming all remaining gas).
   *
   * Requirements:
   * - The divisor cannot be zero.
   */
  function div(uint256 a, uint256 b) internal pure returns (uint256) {
    return div(a, b, "SafeMath: division by zero");
  }

  /**
   * @dev Returns the integer division of two unsigned integers. Reverts with custom message on
   * division by zero. The result is rounded towards zero.
   *
   * Counterpart to Solidity's `/` operator. Note: this function uses a
   * `revert` opcode (which leaves remaining gas untouched) while Solidity
   * uses an invalid opcode to revert (consuming all remaining gas).
   *
   * Requirements:
   * - The divisor cannot be zero.
   */
  function div(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
    // Solidity only automatically asserts when dividing by 0
    require(b > 0, errorMessage);
    uint256 c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold

    return c;
  }

  /**
   * @dev Returns the remainder of dividing two unsigned integers. (unsigned integer modulo),
   * Reverts when dividing by zero.
   *
   * Counterpart to Solidity's `%` operator. This function uses a `revert`
   * opcode (which leaves remaining gas untouched) while Solidity uses an
   * invalid opcode to revert (consuming all remaining gas).
   *
   * Requirements:
   * - The divisor cannot be zero.
   */
  function mod(uint256 a, uint256 b) internal pure returns (uint256) {
    return mod(a, b, "SafeMath: modulo by zero");
  }

  /**
   * @dev Returns the remainder of dividing two unsigned integers. (unsigned integer modulo),
   * Reverts with custom message when dividing by zero.
   *
   * Counterpart to Solidity's `%` operator. This function uses a `revert`
   * opcode (which leaves remaining gas untouched) while Solidity uses an
   * invalid opcode to revert (consuming all remaining gas).
   *
   * Requirements:
   * - The divisor cannot be zero.
   */
  function mod(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
    require(b != 0, errorMessage);
    return a % b;
  }
}
```

## 13.基于solidity0.8.0版本的bep20合约,包含预售功能,及百分之3的燃烧。

`BEP20Token.sol`

```solidity
pragma solidity ^0.8.0;

import "./Context.sol";
import "./IBEP20.sol";
import "./Ownable.sol";
import "./SafeMath.sol";

contract BEP20Token is Context, IBEP20, Ownable {
    using SafeMath for uint256;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    uint256 private _totalSupply;
    uint8 private _decimals;
    string private _symbol;
    string private _name;

    bool private _presaleOpen;
    uint256 private _presaleEndTime;
    uint256 private _presaleRate = 10000000; // 1 BNB = 10000000合约代币
    uint256 private _minPresaleAmount = 20000000000000000; // 最小支持传入0.02 BNB
    uint256 private constant _presaleDuration = 15 days; // 预售持续时间为15天
    uint256 private constant _transactionTaxRate = 3; // 3% transaction tax

    mapping(address => uint256) private presaleRecords; // 记录预售用户及预售金额

    constructor() {
        _name = "TangPing2";
        _symbol = "TPING2";
        _decimals = 6;
        _totalSupply = 10000000000000000;
        _balances[_msgSender()] = _totalSupply;

        emit Transfer(address(0), _msgSender(), _totalSupply);
    }

    function getOwner() external view override returns (address) {
        return owner();
    }

    function decimals() external view override returns (uint8) {
        return _decimals;
    }

    function symbol() external view override returns (string memory) {
        return _symbol;
    }

    function name() external view override returns (string memory) {
        return _name;
    }

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account)
        external
        view
        override
        returns (uint256)
    {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount)
        external
        override
        returns (bool)
    {
        _transfer(_msgSender(), recipient, amount);
        return true;
    }

    function allowance(address owner, address spender)
        external
        view
        override
        returns (uint256)
    {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount)
        external
        override
        returns (bool)
    {
        _approve(_msgSender(), spender, amount);
        return true;
    }

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external override returns (bool) {
        _transfer(sender, recipient, amount);
        uint256 newAllowance = _allowances[sender][_msgSender()].sub(amount);
        require(
            newAllowance >= amount,
            "BEP20: transfer amount exceeds allowance"
        );
        _approve(sender, _msgSender(), newAllowance);
        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue)
        public
        returns (bool)
    {
        _approve(
            _msgSender(),
            spender,
            _allowances[_msgSender()][spender].add(addedValue)
        );
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue)
        public
        returns (bool)
    {
        uint256 newAllowance = _allowances[_msgSender()][spender].sub(
            subtractedValue
        );
        require(
            newAllowance >= subtractedValue,
            "BEP20: decreased allowance below zero"
        );
        _approve(_msgSender(), spender, newAllowance);
        return true;
    }

    function mint(uint256 amount) public onlyOwner returns (bool) {
        _mint(_msgSender(), amount);
        return true;
    }

    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal {
        require(sender != address(0), "BEP20: transfer from the zero address");
        require(recipient != address(0), "BEP20: transfer to the zero address");
        uint256 newBalance = _balances[sender] - amount;
        require(newBalance >= amount, "BEP20: transfer amount exceeds balance");
        uint256 taxAmount = amount.mul(_transactionTaxRate).div(100);
        uint256 transferAmount = amount.sub(taxAmount);
        _balances[sender] = _balances[sender].sub(amount);
        _balances[recipient] = _balances[recipient].add(transferAmount);
        emit Transfer(sender, recipient, transferAmount);

        _burn(sender, taxAmount);
    }

    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "BEP20: mint to the zero address");
        _totalSupply = _totalSupply.add(amount);
        _balances[account] = _balances[account].add(amount);
        emit Transfer(address(0), account, amount);
    }

    function _burn(address account, uint256 amount) internal {
        require(account != address(0), "BEP20: burn from the zero address");
        uint256 newBalance = _balances[account] - amount;
        require(
            newBalance <= _balances[account],
            "BEP20: burn amount exceeds balance"
        );
        _balances[account] = newBalance;
        _totalSupply = _totalSupply.sub(amount);
        emit Transfer(account, address(0), amount);
    }

    function _approve(
        address owner,
        address spender,
        uint256 amount
    ) internal {
        require(owner != address(0), "BEP20: approve from the zero address");
        require(spender != address(0), "BEP20: approve to the zero address");
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }

    function _burnFrom(address account, uint256 amount) internal {
        _burn(account, amount);

        uint256 newAllowance = _allowances[account][_msgSender()] - amount;
        require(
            newAllowance <= _allowances[account][_msgSender()],
            "BEP20: burn amount exceeds allowance"
        );
        _allowances[account][_msgSender()] = newAllowance;

        _approve(account, _msgSender(), newAllowance);
    }

    // 开启预售
    function openPresale() external onlyOwner {
        require(!_presaleOpen, "Presale already open");
        _presaleOpen = true;
        _presaleEndTime = block.timestamp + _presaleDuration;
    }

    // 用户参与预售，向合约发送BNB，合约自动转账相应数量的代币给用户
    function purchasePresale() external payable {
        require(_presaleOpen, "Presale not open");
        require(block.timestamp <= _presaleEndTime, "Presale has ended");
        require(
            msg.value >= _minPresaleAmount,
            "Amount is below minimum presale amount"
        );

        uint256 tokenAmount = msg.value.mul(_presaleRate);
        require(
            tokenAmount <= _balances[owner()],
            "Insufficient presale tokens"
        );

        _balances[owner()] = _balances[owner()].sub(tokenAmount);
        _balances[_msgSender()] = _balances[_msgSender()].add(tokenAmount);
        emit Transfer(owner(), _msgSender(), tokenAmount);

        presaleRecords[_msgSender()] = presaleRecords[_msgSender()].add(
            msg.value
        );
    }

    // 获取预售记录
    function getPresaleRecord(address user) external view returns (uint256) {
        return presaleRecords[user];
    }
}
```

`Context.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/*
 * @dev Provides information about the current execution context, including the
 * sender of the transaction and its data. While these are generally available
 * via msg.sender and msg.data, they should not be accessed in such a direct
 * manner, since when dealing with GSN meta-transactions the account sending and
 * paying for execution may not be the actual sender (as far as an application
 * is concerned).
 *
 * This contract is only required for intermediate, library-like contracts.
 */
contract Context {
  // Empty internal constructor, to prevent people from mistakenly deploying
  // an instance of this contract, which should be used via inheritance.
  constructor () {}

  function _msgSender() internal view returns (address payable) {
    return payable(msg.sender);
  }

  function _msgData() internal view returns (bytes memory) {
    this; // silence state mutability warning without generating bytecode - see https://github.com/ethereum/solidity/issues/2691
    return msg.data;
  }
}
```

`IBEP20.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IBEP20 {
  /**
   * @dev Returns the amount of tokens in existence.
   */
  function totalSupply() external view returns (uint256);

  /**
   * @dev Returns the token decimals.
   */
  function decimals() external view returns (uint8);

  /**
   * @dev Returns the token symbol.
   */
  function symbol() external view returns (string memory);

  /**
  * @dev Returns the token name.
  */
  function name() external view returns (string memory);

  /**
   * @dev Returns the bep token owner.
   */
  function getOwner() external view returns (address);

  /**
   * @dev Returns the amount of tokens owned by `account`.
   */
  function balanceOf(address account) external view returns (uint256);

  /**
   * @dev Moves `amount` tokens from the caller's account to `recipient`.
   *
   * Returns a boolean value indicating whether the operation succeeded.
   *
   * Emits a {Transfer} event.
   */
  function transfer(address recipient, uint256 amount) external returns (bool);

  /**
   * @dev Returns the remaining number of tokens that `spender` will be
   * allowed to spend on behalf of `owner` through {transferFrom}. This is
   * zero by default.
   *
   * This value changes when {approve} or {transferFrom} are called.
   */
  function allowance(address _owner, address spender) external view returns (uint256);

  /**
   * @dev Sets `amount` as the allowance of `spender` over the caller's tokens.
   *
   * Returns a boolean value indicating whether the operation succeeded.
   *
   * IMPORTANT: Beware that changing an allowance with this method brings the risk
   * that someone may use both the old and the new allowance by unfortunate
   * transaction ordering. One possible solution to mitigate this race
   * condition is to first reduce the spender's allowance to 0 and set the
   * desired value afterwards:
   * https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
   *
   * Emits an {Approval} event.
   */
  function approve(address spender, uint256 amount) external returns (bool);

  /**
   * @dev Moves `amount` tokens from `sender` to `recipient` using the
   * allowance mechanism. `amount` is then deducted from the caller's
   * allowance.
   *
   * Returns a boolean value indicating whether the operation succeeded.
   *
   * Emits a {Transfer} event.
   */
  function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

  /**
   * @dev Emitted when `value` tokens are moved from one account (`from`) to
   * another (`to`).
   *
   * Note that `value` may be zero.
   */
  event Transfer(address indexed from, address indexed to, uint256 value);

  /**
   * @dev Emitted when the allowance of a `spender` for an `owner` is set by
   * a call to {approve}. `value` is the new allowance.
   */
  event Approval(address indexed owner, address indexed spender, uint256 value);
}

```

`Ownable.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Context.sol";

/**
 * @dev Contract module which provides a basic access control mechanism, where
 * there is an account (an owner) that can be granted exclusive access to
 * specific functions.
 *
 * By default, the owner account will be the one that deploys the contract. This
 * can later be changed with {transferOwnership}.
 *
 * This module is used through inheritance. It will make available the modifier
 * `onlyOwner`, which can be applied to your functions to restrict their use to
 * the owner.
 */
contract Ownable is Context {
    address private _owner;

    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    /**
     * @dev Initializes the contract setting the deployer as the initial owner.
     */
    constructor() {
        address msgSender = _msgSender();
        _owner = msgSender;
        emit OwnershipTransferred(address(0), msgSender);
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view returns (address) {
        return _owner;
    }

    /**
     * @dev Throws if called by any account other than the owner.
     */
    modifier onlyOwner() {
        require(_owner == _msgSender(), "Ownable: caller is not the owner");
        _;
    }

    /**
     * @dev Leaves the contract without owner. It will not be possible to call
     * `onlyOwner` functions anymore. Can only be called by the current owner.
     *
     * NOTE: Renouncing ownership will leave the contract without an owner,
     * thereby removing any functionality that is only available to the owner.
     */
    function renounceOwnership() public onlyOwner {
        emit OwnershipTransferred(_owner, address(0));
        _owner = address(0);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     * Can only be called by the current owner.
     */
    function transferOwnership(address newOwner) public onlyOwner {
        _transferOwnership(newOwner);
    }

    /**
     * @dev Transfers ownership of the contract to a new account (`newOwner`).
     */
    function _transferOwnership(address newOwner) internal {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        emit OwnershipTransferred(_owner, newOwner);
        _owner = newOwner;
    }
}

```

`SafeMath.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * @dev Wrappers over Solidity's arithmetic operations with added overflow
 * checks.
 *
 * Arithmetic operations in Solidity wrap on overflow. This can easily result
 * in bugs, because programmers usually assume that an overflow raises an
 * error, which is the standard behavior in high level programming languages.
 * `SafeMath` restores this intuition by reverting the transaction when an
 * operation overflows.
 *
 * Using this library instead of the unchecked operations eliminates an entire
 * class of bugs, so it's recommended to use it always.
 */
library SafeMath {
    /**
     * @dev Returns the addition of two unsigned integers, reverting on
     * overflow.
     *
     * Counterpart to Solidity's `+` operator.
     *
     * Requirements:
     * - Addition cannot overflow.
     */
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");

        return c;
    }

    /**
     * @dev Returns the subtraction of two unsigned integers, reverting on
     * overflow (when the result is negative).
     *
     * Counterpart to Solidity's `-` operator.
     *
     * Requirements:
     * - Subtraction cannot overflow.
     */
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        uint256 c = a - b;

        return c;
    }

    /**
     * @dev Returns the multiplication of two unsigned integers, reverting on
     * overflow.
     *
     * Counterpart to Solidity's `*` operator.
     *
     * Requirements:
     * - Multiplication cannot overflow.
     */
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
            return 0;
        }

        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");

        return c;
    }

    /**
     * @dev Returns the integer division of two unsigned integers. Reverts on
     * division by zero. The result is rounded towards zero.
     *
     * Counterpart to Solidity's `/` operator. Note: this function uses a
     * `revert` opcode (which leaves remaining gas untouched) while Solidity
     * uses an invalid opcode to revert (consuming all remaining gas).
     *
     * Requirements:
     * - The divisor cannot be zero.
     */
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        uint256 c = a / b;

        return c;
    }

    /**
     * @dev Returns the remainder of dividing two unsigned integers. (unsigned integer modulo),
     * Reverts with custom message when dividing by zero.
     *
     * Counterpart to Solidity's `%` operator. This function uses a `revert`
     * opcode (which leaves remaining gas untouched) while Solidity uses an
     * invalid opcode to revert (consuming all remaining gas).
     *
     * Requirements:
     * - The divisor cannot be zero.
     */
    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b != 0, "SafeMath: modulo by zero");
        return a % b;
    }
}
```

