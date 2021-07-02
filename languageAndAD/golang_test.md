
全局变量可通过GoStub框架打桩
过程可通过GoStub框架打桩
函数可通过GoStub框架打桩

interface可通过GoMock框架打桩



```
(1) Define an interface that you wish to mock.
      type MyInterface interface {
        SomeMethod(x int64, y string)
      }
(2) Use mockgen to generate a mock from the interface.
(3) Use the mock in a test:
      func TestMyThing(t *testing.T) {
        mockCtrl := gomock.NewController(t)
        defer mockCtrl.Finish()

        mockObj := something.NewMockMyInterface(mockCtrl)
        mockObj.EXPECT().SomeMethod(4, "blah")
        // pass mockObj to a real object and play with it.
```


































