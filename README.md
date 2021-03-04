# Stretchy Header Collection View with images slider

We can add an Stretchy Hader view for collection view simply using "SupplementaryView". 
we can achive this using custom "UICollectionViewFlowLayout" and use the "layoutAttributesForElements" method to update attribute frame base on scroll content offset y.

```swift
  override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        guard let layoutAttributes = super.layoutAttributesForElements(in: rect),
              let collectionView = collectionView else { return nil }
        layoutAttributes.forEach({ (attribute) in
            if attribute.representedElementKind == UICollectionView.elementKindSectionHeader {
                let contentOffsetY = collectionView.contentOffset.y
                if contentOffsetY < 0 {
                    let width = collectionView.frame.width
                    // as contentOffsetY is -ve, the height will increase based on contentOffsetY
                    let height = attribute.frame.height - contentOffsetY
                    attribute.frame = CGRect(x: 0, y: contentOffsetY, width: width, height: height)
                }
            }
        })
        return layoutAttributes
    }
```
 
[Full implemntation here](https://medium.com/@mail2ashislaha/stretchy-header-animation-using-collection-view-custom-layout-f2ce466ec710)

But what if we need add collection view inside the header view?

The client want to replace the fixed product image with an array of products images, so we need to remove the image view and add a collection view with images cells

```swift
    lazy var collectionView: UICollectionView = {
        let layout = UICollectionViewFlowLayout()
        layout.scrollDirection = .horizontal
        let view = UICollectionView(frame: frame, collectionViewLayout: layout)
        view.isPagingEnabled = true
        return view
    }()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        addSubview(collectionView)
        collectionView.translatesAutoresizingMaskIntoConstraints = false
        collectionView.backgroundColor = .blue
        NSLayoutConstraint.activate([
            collectionView.leadingAnchor.constraint(equalTo: leadingAnchor),
            collectionView.topAnchor.constraint(equalTo: topAnchor),
            collectionView.trailingAnchor.constraint(equalTo: trailingAnchor),
            collectionView.bottomAnchor.constraint(equalTo: bottomAnchor)
        ])
        collectionView.dataSource = self
        collectionView.delegate = self
        collectionView.register(AttributeCell.self, forCellWithReuseIdentifier: AttributeCell.cellId)
    }
```

```swift
  extension PDPHeaderView: UICollectionViewDataSource {

      func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
          20
      }

      func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
          let cell = collectionView.dequeueReusableCell(withReuseIdentifier: AttributeCell.cellId, for: indexPath)
          cell.backgroundColor = UIColor(hue: CGFloat(drand48()), saturation: 1, brightness: 1, alpha: 1)
          return cell
      }
  }
```

```swift
  extension PDPHeaderView: UICollectionViewDelegateFlowLayout {

      func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, minimumLineSpacingForSectionAt section: Int) -> CGFloat {
          .zero
      }

      func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, minimumInteritemSpacingForSectionAt section: Int) -> CGFloat {
          .zero
      }

      func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, insetForSectionAt section: Int) -> UIEdgeInsets {
          .zero
      }

      func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize {
          collectionView.frame.size
      }
  }
```


If you try that unfortunately you will end up with this wired behavior

![Mar-05-2021 01-37-10](https://user-images.githubusercontent.com/17902030/110045422-52cf3180-7d53-11eb-95ea-fb150ea7ce43.gif)

Instead of this normal one

![Mar-05-2021 01-38-54](https://user-images.githubusercontent.com/17902030/110045584-91fd8280-7d53-11eb-9bab-14ab47647bf5.gif)

And off course you will find the console start scremaing with warnings.

The problem is when we update the layoutAttribute frame for the header view this will update the collection view frame (the one inside the header view),
but not the images cells. 

so when we are scrolling we see the blue color (the bacground color for the collection view inside the header view) and not the image cell color.

we need to notify the header collection view to update the size for his cells at the same time when we update the layoutAttribute frame for the header view itself.
and we can do that simply using 

```swift
     headerView.collectionView.collectionViewLayout.invalidateLayout()
```
this well invoke all the "UICollectionViewDelegateFlowLayout" and the size well be updated.

So one solition we can do is has an clousre on our custom flow layout class and we can invoke this clousure when we update the header layout attribut frame.


```swift
    typealias UpdateLayoutAttributesObserver = (() -> Void)
    
    var onUpdateLayoutAttributesForSectionHeader: UpdateLayoutAttributesObserver?
    
        override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        guard let layoutAttributes = super.layoutAttributesForElements(in: rect),
              let collectionView = collectionView else { return nil }
        layoutAttributes.forEach({ (attribute) in
            if attribute.representedElementKind == UICollectionView.elementKindSectionHeader {
                let contentOffsetY = collectionView.contentOffset.y
                if contentOffsetY < 0 {
                    let width = collectionView.frame.width
                    // as contentOffsetY is -ve, the height will increase based on contentOffsetY
                    let height = attribute.frame.height - contentOffsetY
                    attribute.frame = CGRect(x: 0, y: contentOffsetY, width: width, height: height)
                }
                onUpdateLayoutAttributesForSectionHeader?()
            }
        })
        return layoutAttributes
    }
```

The client (the view controller on our case) can use this clousre like this.


```swift
        if let layout = collectionViewLayout as? CustomPDPLayout {
            layout.onUpdateLayoutAttributesForSectionHeader = updateHeaderViewLayout()
        }
```

```swift
    private func updateHeaderViewLayout () -> CustomPDPLayout.UpdateLayoutAttributesObserver {
        { [weak self] in
            guard let self = self else { return }
            let firstSection = IndexPath(item: .zero, section: .zero)
            guard let headerView = self.collectionView.supplementaryView(forElementKind: UICollectionView.elementKindSectionHeader, at: firstSection) as? PDPHeaderView else { return }
            headerView.collectionView.collectionViewLayout.invalidateLayout()
        }
    }
```

Now everything will work smoothly.


