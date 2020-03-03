# Container

Java的口号是“write once run anywhere”，但由于Java的产物只被包含了应用，对于Java runtime和特定的运维环境有很多依赖。

相比之下，docker就牛逼了，它的image彻底解决了所有的依赖（一整套的rootfs）...

共用内核-- 环境依赖在rootfs中解决并保持不变性。

